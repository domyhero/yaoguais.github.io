## php cli 执行过程 ##

运行环境

- PHP Version 7.0.0-dev (`./configure  --prefix=/root/php7d --without-pear --enable-fpm --enable-debug`)
- GNU gdb (GDB) Red Hat Enterprise Linux (7.2-75.el6)
- Linux version 2.6.32-504.1.3.el6.x86_64 (gcc version 4.4.7 20120313 (Red Hat 4.4.7-11) (GCC) )

前言：

只要了解过PHP生命周期的应该都知道，php会一次执行MINIT、RINIT、RSHUTDOWN、MSHUTDOWN四个过程，本文就顺着这条线，追踪php在CLI模式下都做了些什么。

目录：

1. 执行main入口函数
2. 执行模块注册流程
3. 模块初始化MINIT流程
4. 请求初始化RINIT流程
5. 请求结束RSHUTDOWN流程
6. 模块销毁MSHUTDOWN函数

### 执行main入口函数 ###

	PHP-SRC/sapi/cli/php_cli.c:1181

	#ifdef PHP_CLI_WIN32_NO_CONSOLE
	int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nShowCmd)
	#else
	int main(int argc, char *argv[])
	#endif
	{
	#ifdef ZTS
		void ***tsrm_ls;
	#endif
	#ifdef PHP_CLI_WIN32_NO_CONSOLE
		int argc = __argc;
		char **argv = __argv;
	#endif

	:1210

	sapi_module_struct *sapi_module = &cli_sapi_module;/*这里将cli_sapi_module结构体赋值给sapi_module*/
	
	:445

	static sapi_module_struct cli_sapi_module = {
		"cli",							/* name */
		"Command Line Interface",    	/* pretty name */	
		php_cli_startup,				/* startup 这里的启动函数指向php_cli_startup函数指针 */
		php_module_shutdown_wrapper,	/* shutdown */	
		NULL,							/* activate */
		sapi_cli_deactivate,			/* deactivate */	
		sapi_cli_ub_write,		    	/* unbuffered write */
		sapi_cli_flush,				    /* flush */
		NULL,							/* get uid */
		NULL,							/* getenv */	
		php_error,						/* error handler */	
		sapi_cli_header_handler,		/* header handler */
		sapi_cli_send_headers,			/* send headers handler */
		sapi_cli_send_header,			/* send header handler */	
		NULL,				            /* read POST data */
		sapi_cli_read_cookies,          /* read Cookies */	
		sapi_cli_register_variables,	/* register server variables */
		sapi_cli_log_message,			/* Log message */
		NULL,							/* Get request time */
		NULL,							/* Child terminate */	
		STANDARD_SAPI_MODULE_PROPERTIES
	};


### 执行模块注册流程 ###

	PHP-SRC/sapi/cli/php_cli.c:1341

	if (sapi_module->startup(sapi_module) == FAILURE)/*在CLI模式下调用php_cli_startup函数*/

	:419

	static int php_cli_startup(sapi_module_struct *sapi_module) /* {{{ */
	{
		if (php_module_startup(sapi_module, NULL, 0)==FAILURE) {/*调用php_module_startup函数*/
			return FAILURE;
		}
		return SUCCESS;
	}

	PHP-SRC/main/main.c:2036

	int php_module_startup(sapi_module_struct *sf, zend_module_entry *additional_modules, uint num_additional_modules)
	{
	/*php使用的一些工具函数*/
	zend_utility_functions zuf;
		//...
	zuf.error_function = php_error_cb;
	zuf.printf_function = php_printf;
	zuf.write_function = php_output_wrapper;
	zuf.fopen_function = php_fopen_wrapper_for_zend;
	zuf.message_handler = php_message_handler_for_zend;
	zuf.block_interruptions = sapi_module.block_interruptions;
	zuf.unblock_interruptions = sapi_module.unblock_interruptions;
	zuf.get_configuration_directive = php_get_configuration_directive_for_zend;
	zuf.ticks_function = php_run_ticks;
	zuf.on_timeout = php_on_timeout;
	zuf.stream_open_function = php_stream_open_for_zend;
	zuf.vspprintf_function = vspprintf;
	zuf.vstrpprintf_function = vstrpprintf;
	zuf.getenv_function = sapi_getenv;
	zuf.resolve_path_function = php_resolve_path_for_zend;
	/*这里的zend_startup比较重要，我们会在最后面分析这个函数*/
	zend_startup(&zuf, NULL);

	:2263
		/*解析php.ini文件*/
		if (php_init_config() == FAILURE) {
			return FAILURE;
		}
		/* startup extensions statically compiled in 注册内核扩展 */
		if (php_register_internal_extensions_func() == FAILURE) {
			php_printf("Unable to start builtin modules\n");
			return FAILURE;
		}
		/* start additional PHP extensions */
		php_register_extensions_bc(additional_modules, num_additional_modules);
		/*注册ini中配置的扩展，即我们开发的第三方扩展*/
		php_ini_register_extensions();

	//:122 PHPAPI int (*php_register_internal_extensions_func)(void) = php_register_internal_extensions;

	PHP-SRC/main/internal_functions_cli.c:88

	PHPAPI int php_register_internal_extensions(void)
	{
		return php_register_extensions(php_builtin_extensions, EXTCOUNT);
	}

	PHP-SRC/main/main.c:1968

	int php_register_extensions(zend_module_entry **ptr, int count)
	{
		zend_module_entry **end = ptr + count;
	
		while (ptr < end) {
			if (*ptr) {
				if (zend_register_internal_module(*ptr)==NULL) {
					return FAILURE;
				}
			}
			ptr++;
		}
		return SUCCESS;
	}

	PHP-SRC/Zend/zend_API.c:1857

	ZEND_API zend_module_entry* zend_register_internal_module(zend_module_entry *module) /* {{{ */
	{
		module->module_number = zend_next_free_module();
		module->type = MODULE_PERSISTENT;
		return zend_register_module_ex(module);
	}
	
	:1797

	ZEND_API zend_module_entry* zend_register_module_ex(zend_module_entry *module) /* {{{ */
	{
		size_t name_len;
		zend_string *lcname;
		zend_module_entry *module_ptr;
	
		if (!module) {
			return NULL;
		}

	//这里我们查看一下zend_module_entry结构体

	PHP-SRC/Zend/zend_modules.h:73

	typedef struct _zend_module_entry zend_module_entry;
	typedef struct _zend_module_dep zend_module_dep;
	
	struct _zend_module_entry {
		unsigned short size;
		unsigned int zend_api;
		unsigned char zend_debug;
		unsigned char zts;
		const struct _zend_ini_entry *ini_entry;
		const struct _zend_module_dep *deps;
		const char *name;
		const struct _zend_function_entry *functions;
		int (*module_startup_func)(INIT_FUNC_ARGS);
		int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);
		int (*request_startup_func)(INIT_FUNC_ARGS);
		int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);
		void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
		const char *version;
		size_t globals_size;
	#ifdef ZTS
		ts_rsrc_id* globals_id_ptr;
	#else
		void* globals_ptr;
	#endif
		void (*globals_ctor)(void *global);
		void (*globals_dtor)(void *global);
		int (*post_deactivate_func)(void);
		int module_started;
		unsigned char type;
		void *handle;
		int module_number;
		const char *build_id;
	};

	//这里我们对比一下swoole扩展的实现
	zend_module_entry swoole_module_entry =
	{
	#if ZEND_MODULE_API_NO >= 20050922
		STANDARD_MODULE_HEADER_EX,
		NULL,
		NULL,
	#else
		STANDARD_MODULE_HEADER,
	#endif
		"swoole",
		swoole_functions,
		PHP_MINIT(swoole),
		PHP_MSHUTDOWN(swoole),
		PHP_RINIT(swoole), //RINIT
		PHP_RSHUTDOWN(swoole), //RSHUTDOWN
		PHP_MINFO(swoole),
	    PHP_SWOOLE_VERSION,
		STANDARD_MODULE_PROPERTIES
	};
	
	PHP-SRC/Zend/zend-API.c:1837

	//继续查看zend_register_module_ex函数的实现
		/*这里将当前的module添加到module_registry这个全局变量中(module_registry即已注册的模块的Hash表)*/
		if ((module_ptr = zend_hash_add_mem(&module_registry, lcname, module, sizeof(zend_module_entry))) == NULL) {
			zend_error(E_CORE_WARNING, "Module '%s' already loaded", module->name);
			zend_string_release(lcname);
			return NULL;
		}
		zend_string_release(lcname);
		module = module_ptr;
		EG(current_module) = module;
		/*这里将扩展中的函数注册到全局函数中去*/
		if (module->functions && zend_register_functions(NULL, module->functions, NULL, module->type)==FAILURE) {
			EG(current_module) = NULL;
			zend_error(E_CORE_WARNING,"%s: Unable to register functions, unable to load", module->name);
			return NULL;
		}
	
		EG(current_module) = NULL;
		return module;
	}


### 模块初始化MINIT流程 ###
	//当执行完register行为后，我们继续回到php_module_startup函数中

	PHP-SRC/main/main.c:2263

	if (php_register_internal_extensions_func() == FAILURE) {
		php_printf("Unable to start builtin modules\n");
		return FAILURE;
	}

	/* start additional PHP extensions */
	php_register_extensions_bc(additional_modules, num_additional_modules);

	//...
	php_ini_register_extensions();
	zend_startup_modules();/*刚才是注册扩展，现在是启动扩展，我们知道启动扩展会执行其MINIT函数*/
	
	PHP-SRC/Zend/zend_API.c:1781

	ZEND_API int zend_startup_modules(void) /* {{{ */
	{
		zend_hash_sort_ex(&module_registry, zend_sort_modules, NULL, 0);
		/*这里调用zend_hash_apply,会回调传入的函数参数zend_startup_module_zval*/
		zend_hash_apply(&module_registry, zend_startup_module_zval);
		return SUCCESS;
	}

	：1669

	static int zend_startup_module_zval(zval *zv) /* {{{ */
	{
		zend_module_entry *module = Z_PTR_P(zv);
		/*这里继续调用zend_startup_module_ex*/
		return zend_startup_module_ex(module);
	}

	:1661

	ZEND_API int zend_startup_module_ex(zend_module_entry *module) /* {{{ */
	{
		size_t name_len;
		zend_string *lcname;
		//...
		/*如果存在启动函数，那么就调用其启动函数*/
		if (module->module_startup_func) {
			EG(current_module) = module;
			if (module->module_startup_func(module->type, module->module_number)==FAILURE) {
				zend_error(E_CORE_ERROR,"Unable to start %s module", module->name);
				EG(current_module) = NULL;
				return FAILURE;
			}
			EG(current_module) = NULL;
		}
		return SUCCESS;
	}
	//至此我们已经找到PHP执行流程的第一个关键点MINIT


### 请求初始化RINIT流程 ###
	
	//现在我们重新回到main函数中

	PHP-SRC/main/main.c:1361

	if (sapi_module->startup(sapi_module) == FAILURE) {
		//...
		exit_status = 1;
		goto out;
	}
	module_started = 1;

	/* -e option */
	if (use_extended_info) {
		CG(compiler_options) |= ZEND_COMPILE_EXTENDED_INFO;
	}

	zend_first_try {
	#ifndef PHP_CLI_WIN32_NO_CONSOLE
			if (sapi_module == &cli_sapi_module) {
	#endif
				/*这里开始执行do_cli进行脚本的解析与执行*/
				exit_status = do_cli(argc, argv);

	PHP-SRC/sapi/cli/php_cli.php:647

	static int do_cli(int argc, char **argv) /* {{{ */
	{
		int c;
		zend_file_handle file_handle;
		int behavior = PHP_MODE_STANDARD;
		//...
		/*调用php_request_startup在请求到来后，解析脚本前，调用模块的RINIT回调函数*/
		if (php_request_startup()==FAILURE) {
			*arg_excp = arg_free;
			fclose(file_handle.handle.fp);
			PUTS("Could not startup.\n");
			goto err;
		}

	PHP-SRC/main/main.c:1586

	int php_request_startup(void)
	{
		int retval = SUCCESS;
		//...
		php_hash_environment();
		/*激活每个模块的RINIT函数*/
		zend_activate_modules();
		PG(modules_activated)=1;


	PHP-SRC/Zend/zend_API.c:2320

	ZEND_API void zend_activate_modules(void) /* {{{ */
	{
		zend_module_entry **p = module_request_startup_handlers;
	
		while (*p) {
			zend_module_entry *module = *p;
			//调用每个模块注册RINIT函数
			if (module->request_startup_func(module->type, module->module_number)==FAILURE) {
				zend_error(E_WARNING, "request_startup() for %s module failed", module->name);
				exit(1);
			}
			p++;
		}
	}
		
	//至此RINIT流程执行完毕,接着回到do_cli函数中


### 请求结束RSHUTDOWN流程 ###

	PHP-SRC/sapi/cli/php_cli.php:982

		switch (behavior) {
		case PHP_MODE_STANDARD:
			if (strcmp(file_handle.filename, "-")) {
				cli_register_file_handles();
			}

			if (interactive && cli_shell_callbacks.cli_shell_run) {
				exit_status = cli_shell_callbacks.cli_shell_run();
			} else {
				//这里执行我们我们的脚本
				php_execute_script(&file_handle);


	PHP-SRC/main/main.c:2471

	PHPAPI int php_execute_script(zend_file_handle *primary_file)
	{
		zend_file_handle *prepend_file_p, *append_file_p;
		//...
		/*执行前置脚本，我们的脚本，以及后置脚本。这里由于没有设置前置后置脚本，所有只有我们的脚本被执行。*/
		retval = (zend_execute_scripts(ZEND_REQUIRE, NULL, 3, prepend_file_p, primary_file, append_file_p) == SUCCESS);


	PHP-SRC/Zend/zend.c:1251

	ZEND_API int zend_execute_scripts(int type, zval *retval, int file_count, ...) /* {{{ */
	{
		va_list files;
		int i;
		zend_file_handle *file_handle;
		zend_op_array *op_array;
	
		va_start(files, file_count);
		/*循环遍历3个脚本*/
		for (i = 0; i < file_count; i++) {
			file_handle = va_arg(files, zend_file_handle *);
			if (!file_handle) {
				continue;
			}
			/*编译脚本生成OPCODE*/
			op_array = zend_compile_file(file_handle, type);
			if (file_handle->opened_path) {
				zend_hash_str_add_empty_element(&EG(included_files),
				 file_handle->opened_path, strlen(file_handle->opened_path));
			}
			zend_destroy_file_handle(file_handle);
			if (op_array) {
				//执行OPCODE
				zend_execute(op_array, retval);
				zend_exception_restore();
				if (EG(exception)) {
					/*如果产生异常，那么调用用户注册的异常捕捉函数*/
					if (Z_TYPE(EG(user_exception_handler)) != IS_UNDEF) {
						zval orig_user_exception_handler;
						zval params[1], retval2;
						zend_object *old_exception;
						old_exception = EG(exception);
						EG(exception) = NULL;
						ZVAL_OBJ(&params[0], old_exception);
						ZVAL_COPY_VALUE(&orig_user_exception_handler, &EG(user_exception_handler));
						ZVAL_UNDEF(&retval2);
						if (call_user_function_ex(CG(function_table), NULL,
							 &orig_user_exception_handler, &retval2, 1, params, 1, NULL) == SUCCESS) {
							zval_ptr_dtor(&retval2);
							if (EG(exception)) {
								OBJ_RELEASE(EG(exception));
								EG(exception) = NULL;
							}
							OBJ_RELEASE(old_exception);
						} else {
							EG(exception) = old_exception;
							zend_exception_error(EG(exception), E_ERROR);
						}
					} else {
						zend_exception_error(EG(exception), E_ERROR);
					}
				}
				destroy_op_array(op_array);
				efree_size(op_array, sizeof(zend_op_array));
			} else if (type==ZEND_REQUIRE) {
				va_end(files);
				return FAILURE;
			}
		}
		va_end(files);
	
		return SUCCESS;
	}
	//至此脚本执行完成


	PHP-SRC/sapi/cli/php_cli.c:1158

	if (request_started) {
		/*调用RSHUTDOWN函数*/
		php_request_shutdown((void *) 0);
	}

	PHP-SRC/main/main.c:1825

	/* 5. Call all extensions RSHUTDOWN functions */
	if (PG(modules_activated)) {
		zend_deactivate_modules();
		php_free_shutdown_functions();
	}

	PHP-SRC/Zend/zenc_API.c:2351

	ZEND_API void zend_deactivate_modules(void) /* {{{ */
	{
		EG(current_execute_data) = NULL; /* we're no longer executing anything */
	
		zend_try {
			if (EG(full_tables_cleanup)) {
				zend_hash_reverse_apply(&module_registry, module_registry_cleanup);
			} else {
				zend_module_entry **p = module_request_shutdown_handlers;
	
				while (*p) {
					zend_module_entry *module = *p;
					/*调用用户注册的RSHUTDOWN函数*/
					module->request_shutdown_func(module->type, module->module_number);
					p++;
				}
			}
		} zend_end_try();
	}

	//至此RSHUTDOWN流程执行完毕

### 模块销毁MSHUTDOWN函数 ###

	继续回到main函数中,分析MSHUTDOWN执行的位置,MSHUTDOWN隐藏的比较深。

	PHP-SRC/sapi/cli/php_cli.c:1376

		if (module_started) {
			/*调用php的模块关闭函数*/
			php_module_shutdown();
		}
		if (sapi_started) {
			sapi_shutdown();
		}
	#ifdef ZTS
		tsrm_shutdown();
	#endif
	
		/*
		 * Do not move this de-initialization. It needs to happen right before
		 * exiting.
		 */
		cleanup_ps_args(argv);
		exit(exit_status);
	}

	:2408

	void php_module_shutdown(void)
	{
		//...	
		sapi_flush();
		/*继续调用zend_shutdown函数*/
		zend_shutdown();

	
	PHP-SRC/Zend/zend.c:736

	void zend_shutdown(void) /* {{{ */
	{
	#ifdef ZEND_SIGNALS
		zend_signal_shutdown();
	#endif
	//...
	/*销毁模块*/
	zend_destroy_modules();

	PHP-SRC/Zend/zend_API.c:1789

	ZEND_API void zend_destroy_modules(void) /* {{{ */
	{
		free(class_cleanup_handlers);
		free(module_request_startup_handlers);
		/*这里调用zend_hash_graceful_reverse_destroy销毁module_registry这个全局Hash表*/
		zend_hash_graceful_reverse_destroy(&module_registry);
	}
	
	PHP-SRC/Zend/zend_hash.c:1037

	ZEND_API void zend_hash_graceful_reverse_destroy(HashTable *ht)
	{
		uint32_t idx;
		Bucket *p;
	
		IS_CONSISTENT(ht);
	
		idx = ht->nNumUsed;
		while (idx > 0) {
			idx--;
			p = ht->arData + idx;
			if (Z_TYPE(p->val) == IS_UNDEF) continue;
			/*遍历Hash表 函数每个元素*/
			_zend_hash_del_el(ht, idx, p);
		}
	
		if (ht->u.flags & HASH_FLAG_INITIALIZED) {
			pefree(ht->arData, ht->u.flags & HASH_FLAG_PERSISTENT);
		}
	
		SET_INCONSISTENT(HT_DESTROYED);
	}

	:657

	static zend_always_inline void _zend_hash_del_el(HashTable *ht, uint32_t idx, Bucket *p)
	{
		Bucket *prev = NULL;
		
		//...
		_zend_hash_del_el_ex(ht, idx, p, prev);
	}
	
	:615

	static zend_always_inline void _zend_hash_del_el_ex(HashTable *ht, uint32_t idx, Bucket *p, Bucket *prev)
	{
		//...
		/*为每一个元素调用Hash表注册的析构函数*/
		if (ht->pDestructor) {
			zval tmp;
			ZVAL_COPY_VALUE(&tmp, &p->val);
			ZVAL_UNDEF(&p->val);
			ht->pDestructor(&tmp);
		} else {
			ZVAL_UNDEF(&p->val);
		}
		HANDLE_UNBLOCK_INTERRUPTIONS();
	}

	/*想起我们前面提到php_module_startup中的zend_startup函数,它便为module_registry注册了析构函数*/

	PHP-SRC/Zend/zend.c:559

	int zend_startup(zend_utility_functions *utility_functions, char **extensions) /* {{{ */
	{
		//...
		zend_hash_init_ex(GLOBAL_FUNCTION_TABLE, 1024, NULL, ZEND_FUNCTION_DTOR, 1, 0);
		zend_hash_init_ex(GLOBAL_CLASS_TABLE, 64, NULL, ZEND_CLASS_DTOR, 1, 0);
		zend_hash_init_ex(GLOBAL_AUTO_GLOBALS_TABLE, 8, NULL, auto_global_dtor, 1, 0);
		zend_hash_init_ex(GLOBAL_CONSTANTS_TABLE, 128, NULL, ZEND_CONSTANT_DTOR, 1, 0);
		//注册析构函数为module_destructor_zval
		zend_hash_init_ex(&module_registry, 32, NULL, module_destructor_zval, 1, 0);
	
	:533

	static void module_destructor_zval(zval *zv) /* {{{ */
	{
		zend_module_entry *module = (zend_module_entry*)Z_PTR_P(zv);
		//调用模块的析构函数
		module_destructor(module);
		free(module);
	}
	
	PHP-SRC/Zend/zend_API.c:2276

	void module_destructor(zend_module_entry *module) /* {{{ */
	{
	
		if (module->type == MODULE_TEMPORARY) {
			zend_clean_module_rsrc_dtors(module->module_number);
			clean_module_constants(module->module_number);
			clean_module_classes(module->module_number);
		}
	
		if (module->module_started && module->module_shutdown_func) {
			//...
			//执行RSHUTDOWN函数
			module->module_shutdown_func(module->type, module->module_number);
		}
	
	/*至此MSHUTDOWN流程执行完毕，并且main函数随后也执行完毕*/