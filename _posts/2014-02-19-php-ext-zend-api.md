---
layout: post
title: "PHP Ext API"
description: ""
category: PHP
tags: [PHP扩展]
---
{% include JB/setup %}

#### 关于本函数列表
- - -
为了帮助php开发者更方便的开发扩展，这里整理了一份ZEND API的函数列表，由于本人能力有限，这些并不是全部，我打算在git上开个项目，来撰写这些API的使用范例，如果你对这个项目感兴趣，并想贡献一份力量，请发email给我**yangrokety@gmail.com**

#### 输出相关 About Output
- - -
1. `php_printf(format, var)`
2. `PHPWRITE(string, strlen(string))`
3. `spprintf(char *, length, format, char *)`：使用该函数前必须先分配好第一个参数的内存空间
4. `snprintf(char **, length, char *)`：会动态地为第一个参数分配内存

上述3、4函数详细讨论：[Difference between sprintf, snprintf and spprintf](http://osdir.com/ml/php.devel/2002-06/msg00840.html)

<!--more-->
#### 参数相关 About Parameters
- - -
1. `int zend_parse_parameters(int num_args TSRMLS_DC, char *type_spec, ...)`

函数负责读取用户从参数堆栈传递来参数，并将其适当地转换后放入局部C语言变量。如果用户传递的参数个数有误或类型不可被转换，函数会发出一个冗长的错误信息，并返回FAILURE。
第一个参数应该为`ZEND_NUM_ARGS()`。

类型说明符和变量存储结构：
![type](/assets/img/201402190101.png)
`|`：在该符号之后的参数都是可选的
`/`：在该字符之后的变量如果不是通过引用传递，那么对其进行ZVAL分离，即执行`SEPARATE_ZVAL_IF_NOT_REF()`
`!`：在该字符之后的变量可以为特定的类型或者NULL（对于上图中的说明符，除了'b'，'l'，'d'都适用），如果传递了NULL，那么指针将会指向NULL
具体参数解析例子，可以查看PHP源代码`phpsrc/README.PARAMETER_PARSING_API`

#### 返回值相关 About Return Values
- - -
1. `RETURN_NULL()`
2. `RETUTN_BOOL(b)` b: 0 => FALSE, non-0 => TRUE
3. `RETUTN_TRUE`
4. `RETUTN_FALSE`
5. `RETUTN_LONG(l)` l: Integer value
6. `RETUTN_DOUBLE(d)` d: Floating point value
7. `RETURN_STRING(str, dup)` str: char* string value dup: 0/1 flag, duplicate string?
8. `RETURN_STRINGL(str, len, dup)` len: Predetermined string length
9. `RETURN_EMPTY_STRING()`

#### 数组相关 Aobut Array
- - -
关联数组 Associate Array：
1. `int add_assoc_long(zval *arg, char *key, long n)`
2. `add_assoc_null(zval *arg, char *key)`
3. `add_assoc_bool(zval *arg, char *key, int b)`
4. `add_assoc_resource(zval *arg, char *key, int r)`
5. `add_assoc_double(zval *arg, char *key, double d)`
6. `add_assoc_string(zval *arg, char *key, char *str, int dup)`
7. `int add_assoc_stringl(zval *arg, char *key, char *str, uint len, int dup);`
8. `int add_assoc_zval(zval *arg, char *key, zval *value)`

索引数组 Index Array：
1. `int add_index_long(zval *arg, ulong idx, long n)`
2. `int add_index_null(zval *arg, ulong idx)`
3. `int add_index_bool(zval *arg, ulong idx, int b)`
4. `int add_index_resource(zval *arg, ulong idx, int r)`
5. `int add_index_double(zval *arg, ulong idx, double d)`
6. `int add_index_string(zval *arg, ulong idx, const char *str, int duplicate)`
7. `int add_index_stringl(zval *arg, ulong idx, const char *str, uint length, int duplicate)`
8. `int add_index_zval(zval *arg, ulong index, zval *value)`
9. `int add_next_index_long(zval *arg, long n)`
10. `int add_next_index_null(zval *arg)`
11. `int add_next_index_bool(zval *arg, int b)`
12. `int add_next_index_resource(zval *arg, int r)`
13. `int add_next_index_double(zval *arg, double d)`
14. `int add_next_index_string(zval *arg, const char *str, int duplicate)`
15. `int add_next_index_stringl(zval *arg, const char *str, uint length, int duplicate)`
16. `int add_next_index_zval(zval *arg, zval *value)`

#### Hash Table
- - -
startup/shutdown：
1. `int zend_hash_init(HashTable *ht,uint nSize, hash_func_t pHashFunction, dtor_func_t pDestructor, zend_bool persistent);`
2. `void zend_hash_destroy(HashTable *ht)`
3. `void zend_hash_clean(HashTable *ht)` Removes all elements from the HashTable

additions/updates/changes：
1. `int zend_hash_update(HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest)`
2. `int zend_hash_add(HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest)`
2. `ulong zend_get_hash_value(char *arKey, uint nKeyLength)`
3. `int zend_hash_num_elements(HashTable *ht)` count($ht)
4. `int zend_hash_quick_add(HashTable *ht, const char *arKey, uint nKeyLength, ulong h, void *pData, uint nDataSize, void **pDest)`
5. `int zend_hash_quick_update(HashTable *ht, const char *arKey, uint nKeyLength, ulong h, void *pData, uint nDataSize, void **pDest)`
6. `int zend_hash_index_update(HashTable *ht, ulong h, void *pData, uint nDataSize, void **pDest)`
7. `int zend_hash_next_index_insert(HashTable *ht, void *pData, uint nDataSize, void **pDest)`
8. `int zend_hash_add_empty_element(HashTable *ht, const char *arKey, uint nKeyLength);`
10. `void zend_hash_graceful_destroy(HashTable *ht)`
11. `void zend_hash_graceful_reverse_destroy(HashTable *ht)`
12. `void zend_hash_apply(HashTable *ht, apply_func_t apply_func TSRMLS_DC)`
13. `void zend_hash_apply_with_argument(HashTable *ht, apply_func_arg_t apply_func, void * TSRMLS_DC)`
14. `void zend_hash_apply_with_arguments(HashTable *ht TSRMLS_DC, apply_func_args_t apply_func, int, ...)`
15. `void zend_hash_reverse_apply(HashTable *ht, apply_func_t apply_func TSRMLS_DC);`

Deletes：
1. `int zend_hash_del(HashTable *ht, const char *arKey, uint nKeyLength)`
2. `int zend_hash_quick_del(HashTable *ht, const char *arKey, uint nKeyLength, ulong h)`
3. `zend_hash_index_del(HashTable *ht, ulong h)`

Data retreival：
1. `int zend_hash_find(const HashTable *ht, const char *arKey, uint nKeyLength, void **pData)`
2. `int zend_hash_index_find(const HashTable *ht, ulong h, void **pData)`
3. `int zend_hash_quick_find(const HashTable *ht, const char *arKey, uint nKeyLength, ulong h, void **pData);`

Misc：
1. `int zend_hash_exists(const HashTable *ht, const char *arKey, uint nKeyLength)`
2. `int zend_hash_quick_exists(const HashTable *ht, const char *arKey, uint nKeyLength, ulong h)`
3. `int zend_hash_index_exists(const HashTable *ht, ulong h)`
4. `ulong zend_hash_next_free_element(const HashTable *ht)`

traversing：
1. `int zend_hash_has_more_elements(ht)`
2. `int zend_hash_move_forward(ht)`
3. `int zend_hash_move_backwards(ht)`
4. `int zend_hash_get_current_key(ht, str_index, num_index, duplicate)`
5. `int zend_hash_get_current_key_type(ht)`
6. `int zend_hash_get_current_data(ht, pData)`
7. `void zend_hash_internal_pointer_reset(ht)`
8. `void zend_hash_internal_pointer_end(ht)`
9. `int zend_hash_update_current_key(ht, key_type, str_index, str_length, num_index)`

Copying, merging and sorting：
1. `void zend_hash_copy(HashTable *target, HashTable *source, copy_ctor_func_t pCopyConstructor, void *tmp, uint size)`
2. ` void _zend_hash_merge(HashTable *target, HashTable *source, copy_ctor_func_t pCopyConstructor, void *tmp, uint size, int overwrite ZEND_FILE_LINE_DC)`
3. `void zend_hash_merge_ex(HashTable *target, HashTable *source, copy_ctor_func_t pCopyConstructor, uint size, merge_checker_func_t pMergeSource, void *pParam)`
4. `int zend_hash_sort(HashTable *ht, sort_func_t sort_func, compare_func_t compare_func, int renumber TSRMLS_DC)`
5. `int zend_hash_compare(HashTable *ht1, HashTable *ht2, compare_func_t compar, zend_bool ordered TSRMLS_DC)`
6. `int zend_hash_minmax(const HashTable *ht, compare_func_t compar, int flag, void **pData TSRMLS_DC);`

debug：
1. `void zend_hash_display_pListTail(const HashTable *ht)`
2. `void zend_hash_display(const HashTable *ht)`

#### About Zval
- - -
accessing a zval：
1. `(zval).value.lval Z_LVAL(zval)`
2. `((zend_bool)(zval).value.lval) Z_BVAL(zval)`
3. `(zval).value.dval Z_DVAL(zval)`
4. `(zval).value.str.val Z_STRVAL(zval)`
5. `(zval).value.str.len Z_STRLEN(zval)`
6. `(zval).value.ht Z_ARRVAL(zval)`
7. `(zval).value.obj Z_OBJVAL(zval)`
8. `Z_OBJVAL(zval).handle Z_OBJ_HANDLE(zval)`
9. `Z_OBJVAL(zval).handlers Z_OBJ_HT(zval)`
10. `zend_class_entry* Z_OBJCE(zval)`
11. `HashTable* Z_OBJPROP(zval)`
12. `Z_OBJ_HT((zval))->hf Z_OBJ_HANDLER(zval, hf)`
13. `(zval).value.lval Z_RESVAL(zval)`
14. `Z_OBJDEBUG(zval,is_tmp)`
15. `(zval).type Z_TYPE(zval)`
16. `Z_*_P(zval_p)` 1-15重复一遍 相当于 `Z_*(*zval)`
17. `Z_*_PP(zval_pp)` 1-15重复一遍 相当于 `Z_*(**zval)`

reference count and is-ref：
1. `Z_REFCOUNT(z)` Retrieve reference count
2. `Z_SET_REFCOUNT(z, rc)` Set reference count to rc
3. `Z_ADDREF(z)` Increment reference count
4. `Z_DELREF(z)` Decrement reference count
5. `Z_ISREF(z)` Whether zval is a reference
6. `Z_SET_ISREF(z)` Makes zval a reference variable
7. `Z_UNSET_ISREF(z)` Resets the is-reference flag
8. `Z_SET_ISREF_TO(z, isref)` Make zval a reference is isref != 0
9. `Z_*_P(zval_p)` 1-8重复一遍 相当于 `Z_*(*zval)`
10. `Z_*_PP(zval_pp)` 1-8重复一遍 相当于 `Z_*(**zval)`

setting types and values：
1. `ZVAL_RESOURCE(z, l)`
2. `ZVAL_BOOL(z, b)`
3. `ZVAL_NULL(z)`
4. `ZVAL_LONG(z, l)`
5. `ZVAL_DOUBLE(z, d)`
6. `ZVAL_STRING(z, s, duplicate)`
7. `ZVAL_STRINGL(z, s, l, duplicate)`
8. `ZVAL_EMPTY_STRING(z)`
9. `ZVAL_ZVAL(z, zv, copy, dtor)`

allocate and initialize a zval：
1. `INIT_PZVAL(zp)` Set reference count and isref 0
2. `INIT_ZVAL(z)` Initialize and set NULL, no pointer
3. `ALLOC_INIT_ZVAL(zp)` Allocate and initialize a zval
4. `MAKE_STD_ZVAL(zv)` Allocate, initialize and set NULL
5. `PZVAL_IS_REF(z)`
6. `ZVAL_COPY_VALUE(z, v)`
7. `ZVAL_COPY_VALUE(z, v)`
8. `INIT_PZVAL_COPY(z, v)`
9. `SEPARATE_ZVAL(ppzv)`
10. `SEPARATE_ZVAL_IF_NOT_REF(ppzv)`
11. `SEPARATE_ZVAL_TO_MAKE_IS_REF(ppzv)`
12. `COPY_PZVAL_TO_ZVAL(zv, pzv)`
13. `MAKE_COPY_ZVAL(ppzv, pzv)`
14. `REPLACE_ZVAL_VALUE(ppzv_dest, pzv_src, copy)`
15. `SEPARATE_ARG_IF_REF(varptr)`
16. `READY_TO_DESTROY(zv)`

#### 内存分配相关 About Memory Allocation
- - -
1. `viod* emalloc(size_t size)`
2. `void* ecalloc(size_t nmemb, size_t size)`
3. `void* erealloc(void *ptr, size_t size)`
4. `void* estrdup(char *str)`
5. `void* estrndup(char *str, size_t len)`
6. `void efree(void *ptr)`
7. `void *safe_emalloc(size_t nmemb, size_t size, size_t adtl)`
8. `void *STR_EMPTY_ALLOC(void)`
9. `void* pemalloc(size_t size, int persist)`
10. `void* pecalloc(size_t nmemb, size_t size, int persist)`
11. `void* perealloc(void *ptr, size_t size, int persist)`
12. `void* pestrdup(char *str, int persist)`
13. `void pefree(void *ptr, int persist)`
14. `void* safe_pemalloc(size_t nmemb, size_t size, size_t addtl, int persist)`

#### 常量相关 About Constants
- - -
flags：
1. `#define CONST_CS             (1<<0)              /* Case Sensitive */`
2. `#define CONST_PERSISTENT        (1<<1)              /* Persistent */`
3. `#define CONST_CT_SUBST          (1<<2)              /* Allow compile-time substitution */`

register：
1. `REGISTER_LONG_CONSTANT(name, lval, flags)`
2. `REGISTER_DOUBLE_CONSTANT(name, dval, flags)`
3. `REGISTER_STRING_CONSTANT(name, str, flags)`
4. `REGISTER_STRINGL_CONSTANT(name, str, len, flags)`
5. `REGISTER_NS_LONG_CONSTANT(ns, name, lval, flags)`
6. `REGISTER_NS_DOUBLE_CONSTANT(ns, name, dval, flags)`
7. `REGISTER_NS_STRING_CONSTANT(ns, name, str, flags)`
8. `REGISTER_NS_STRINGL_CONSTANT(ns, name, str, len, flags)`
9. `REGISTER_MAIN_LONG_CONSTANT(name, lval, flags)`
10. `REGISTER_MAIN_DOUBLE_CONSTANT(name, dval, flags)`
11. `REGISTER_MAIN_STRING_CONSTANT(name, str, flags)`
12. `REGISTER_MAIN_STRINGL_CONSTANT(name, str, len, flags)`
13. `int zend_register_constant(zend_constant *c TSRMLS_DC)`

get：
1. `int zend_get_constant(const char *name, uint name_len, zval *result TSRMLS_DC)`
2. `zend_constant *zend_quick_get_constant(const zend_literal *key, ulong flags TSRMLS_DC)`
3. `zend_constant *zend_quick_get_constant(const zend_literal *key, ulong flags TSRMLS_DC);`

copy：
1. `void zend_copy_constants(HashTable *target, HashTable *sourc)`
2. `void copy_zend_constant(zend_constant *c)`

#### 对象相关 About Object
- - -
init：
1. `INIT_CLASS_ENTRY(class_container, class_name, functions)`
2. `INIT_CLASS_ENTRY_INIT_METHODS(class_container, functions, handle_fcall, handle_propget, handle_propset, handle_propunset, handle_propisset)`
3. `INIT_OVERLOADED_CLASS_ENTRY(class_container, class_name, functions, handle_fcall, handle_propget, handle_propset)`
4. `INIT_NS_CLASS_ENTRY(class_container, ns, class_name, functions)`
5. `INIT_OVERLOADED_NS_CLASS_ENTRY(class_container, ns, class_name, functions, handle_fcall, handle_propget, handle_propset)`
6. `int object_init(arg)`
7. `int object_and_properties_init(arg, ce, properties)`
8. `void zend_object_std_init(zend_object *object, zend_class_entry *ce TSRMLS_DC)`
9. `zend_object_value zend_objects_new(zend_object **object, zend_class_entry *class_type TSRMLS_DC)`
10. `void zend_object_store_set_object(zval *zobject, void *object TSRMLS_DC)`
11. `void zend_object_store_ctor_failed(zval *zobject TSRMLS_DC)`
12. `zval *zend_object_create_proxy(zval *object, zval *member TSRMLS_DC)`

register：
1. `zend_class_entry *zend_register_internal_class(zend_class_entry *class_entry TSRMLS_DC)`
2. `zend_class_entry *zend_register_internal_interface(zend_class_entry *orig_class_entry TSRMLS_DC)`
3. `int zend_register_class_alias(name, ce)`
4. `int zend_register_ns_class_alias(ns, name, ce)`

declaring class constants：
1. `int zend_declare_class_constant(zend_class_entry *ce, const char *name, size_t name_length, zval *value TSRMLS_DC)`
2. `int zend_declare_class_constant_null(zend_class_entry *ce, const char *name, size_t name_length TSRMLS_DC)`
3. `int zend_declare_class_constant_long(zend_class_entry *ce, const char *name, size_t name_length, long value TSRMLS_DC)`
4. `int zend_declare_class_constant_bool(zend_class_entry *ce, const char *name, size_t name_length, zend_bool value TSRMLS_DC)`
5. `int zend_declare_class_constant_double(zend_class_entry *ce, const char *name, size_t name_length, double value TSRMLS_DC)`
6. `int zend_declare_class_constant_stringl(zend_class_entry *ce, const char *name, size_t name_length, const char *value, size_t value_length TSRMLS_DC)`
7. `int zend_declare_class_constant_string(zend_class_entry *ce, const char *name, size_t name_length, const char *value TSRMLS_DC)`

implements：
1. `void zend_class_implements(zend_class_entry *class_entry TSRMLS_DC, int num_interfaces, ...)`

declaring property：
1. . `int zend_declare_property(zend_class_entry *ce, const char *name, int name_length, zval *property, int access_type TSRMLS_DC)`
2. `int zend_declare_property_ex(zend_class_entry *ce, const char *name, int name_length, zval *property, int access_type, const char *doc_comment, int doc comment_len TSRMLS_DC)`
3. `int zend_declare_property_null(zend_class_entry *ce, const char *name, int name_length, int access_type TSRMLS_DC)`
4. `int zend_declare_property_bool(zend_class_entry *ce, const char *name, int name_length, long value, int access_type TSRMLS_DC)`
5. `int zend_declare_property_long(zend_class_entry *ce, const char *name, int name_length, long value, int access_type TSRMLS_DC)`
6. `int zend_declare_property_double(zend_class_entry *ce, const char *name, int name_length, double value, int access_type TSRMLS_DC)`
7. `int zend_declare_property_string(zend_class_entry *ce, const char *name, int name_length, const char *value, int access_type TSRMLS_DC)`
8. `int zend_declare_property_stringl(zend_class_entry *ce, const char *name, int name_length, const char *value, int value_len, int access_type TSRMLS_DC)`

update：
1. `void zend_update_class_constants(zend_class_entry *class_type TSRMLS_DC)`
2. `void zend_update_property(zend_class_entry *scope, zval *object, const char *name, int name_length, zval *value TSRMLS_DC)`
3. `void zend_update_property_null(zend_class_entry *scope, zval *object, const char *name, int name_length TSRMLS_DC)`
4. `void zend_update_property_bool(zend_class_entry *scope, zval *object, const char *name, int name_length, long value TSRMLS_DC)`
5. `void zend_update_property_long(zend_class_entry *scope, zval *object, const char *name, int name_length, long value TSRMLS_DC)`
6. `void zend_update_property_double(zend_class_entry *scope, zval *object, const char *name, int name_length, double value TSRMLS_DC)`
7. `void zend_update_property_string(zend_class_entry *scope, zval *object, const char *name, int name_length, const char *value TSRMLS_DC)`
8. `void zend_update_property_stringl(zend_class_entry *scope, zval *object, const char *name, int name_length, const char *value, int value_length TSRMLS DC)`
9. `int zend_update_static_property(zend_class_entry *scope, const char *name, int name_length, zval *value TSRMLS_DC)`
10. `int zend_update_static_property_null(zend_class_entry *scope, const char *name, int name_length TSRMLS_DC)`
11. `int zend_update_static_property_bool(zend_class_entry *scope, const char *name, int name_length, long value TSRMLS_DC)`
12. `int zend_update_static_property_long(zend_class_entry *scope, const char *name, int name_length, long value TSRMLS_DC)`
13. `int zend_update_static_property_double(zend_class_entry *scope, const char *name, int name_length, double value TSRMLS_DC)`
14. `int zend_update_static_property_string(zend_class_entry *scope, const char *name, int name_length, const char *value TSRMLS_DC)`
15. `int zend_update_static_property_stringl(zend_class_entry *scope, const char *name, int name_length, const char *value, int value_length TSRMLS_DC)`

read：
1. `zval *zend_read_property(zend_class_entry *scope, zval *object, const char *name, int name_length, zend_bool silent TSRMLS_DC)`
2. `zval *zend_read_static_property(zend_class_entry *scope, const char *name, int name_length, zend_bool silent TSRMLS_DC)`
3. `zend_class_entry *zend_get_class_entry(const zval *zobject TSRMLS_DC)`
4. `int zend_get_object_classname(const zval *object, const char **class_name, zend_uint *class_name_len TSRMLS_DC)`
5. `char *zend_get_type_by_const(int type)`
6. `getThis()`
7. `zend_object *zend_objects_get_address(const zval *object TSRMLS_DC)`
8. `void *zend_object_store_get_object(const zval *object TSRMLS_DC)`
9. `void *zend_object_store_get_object_by_handle(zend_object_handle handle TSRMLS_DC)`

copy and free：
1. `void zend_objects_clone_members(zend_object *new_object, zend_object_value new_obj_val, zend_object *old_object, zend_object_handle handle TSRMLS_DC)`
2. `zend_object_value zend_objects_clone_obj(zval *object TSRMLS_DC)`
3. `void zend_objects_free_object_storage(zend_object *object TSRMLS_DC)`
4. `zend_object_value zend_objects_store_clone_obj(zval *object TSRMLS_DC)`

destruct：
1. `void zend_object_std_dtor(zend_object *object TSRMLS_DC)`
2. `void zend_objects_destroy_object(zend_object *object, zend_object_handle handle TSRMLS_DC)`
3. `void zend_objects_store_call_destructors(zend_objects_store *objects TSRMLS_DC)`
4. `void zend_objects_store_mark_destructed(zend_objects_store *objects TSRMLS_DC)`
5. `void zend_objects_store_destroy(zend_objects_store *objects)`
6. `void zend_objects_store_free_object_storage(zend_objects_store *objects TSRMLS_DC)`

refcount and is-ref：
1. `void zend_objects_store_add_ref(zval *object TSRMLS_DC)`
2. `void zend_objects_store_del_ref(zval *object TSRMLS_DC)`
3. `void zend_objects_store_add_ref_by_handle(zend_object_handle handle TSRMLS_DC)`
4. `zend_uint zend_objects_store_get_refcount(zval *object TSRMLS_DC)`