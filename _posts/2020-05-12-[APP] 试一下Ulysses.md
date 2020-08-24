这篇文章主要目的是为了试试Ulysses在这个github.io上好不好用。至于写的东西嘛，就写写今天刚研究的cJSON的东西吧——就算是凑字数了。
至于CS:APP第四章，我看到之前的计划里没列这个，设计处理器离我也的确比较遥远；所以，第四章就先跳过去，之后从第五章接着看，讲的是程序优化，似乎非常有用。
好了，下面记录一下cJSON。

## cJSON简介
简单来说就是用纯C语言写的处理JSON文档的库，使用起来挺方便的，可以直接将cJSON.c和cJSON.h文件复制到自己的项目代码中使用，其许可证也是对商业友好的。
链接：[cJSON Github][1]

## 关于例程
在网上找了cJSON的使用说明。其实最好的说明就是Github上的readme.mk了，不过别人的博客写的会更为直接点。
我参考的是[Rotation.的一篇博客][2]，然后也自己试了试人家写的程序，并做了一点说明。
对这篇博客一些内容，有几点我不赞同或者没有说明白：
1. 我认为编译JSON不用链接math库
2. s = cJSON\_Parse(cJSON);在s使用完后，必须free(s);
3. 对于整个JSON信息，只需对root进行删除操作。
4. 删除操作cJSON\_Delete(NULL)是安全的。
5. 对于json数组里又是JSON对象，作者说要把这个项先转为普通字符串，再转为json对象。我不这么认为，虽然我也没测试，我想不用再经过字符串，可以直接当做json对象解析。

## 例程1
创建JSON，并添加一些信息。
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "cJSON.h"

int main(int argc, char *argv[])
{
	cJSON *user = NULL;
	char *out = NULL;

	user = cJSON_CreateObject();
	cJSON_AddStringToObject(user, "name", "aningsk");
	cJSON_AddStringToObject(user, "passwd", "123456");
	cJSON_AddNumberToObject(user, "number", 1);

	out = cJSON_Print(user);
	printf("%s\n", out);

	free(out); // After use @out, I think it's necessary to free it.
	cJSON_Delete(user);

	return 0;
}

```
这里需要注意的是：
1. 使用cJSON\_Print()后，需要对那个指针使用free()。

## 例程2
创建JSON数组，数组里一个字符串、一个数字。
```c
#include <stdio.h>
#include <stdlib.h>
#include "cJSON.h"

int create_js(void)
{
	cJSON *root = NULL;
	cJSON *js_body = NULL;
	char *string = NULL;

	root = cJSON_CreateArray();
	if (NULL == root) {
		printf("Error: create root\n");
		return -1;
	}
	cJSON_AddItemToArray(root, cJSON_CreateString("Hello world"));
	cJSON_AddItemToArray(root, cJSON_CreateNumber(10));

	string = cJSON_PrintUnformatted(root);
	if (NULL != string) {
		printf("%s\n", string);
		free(string);
	}

	cJSON_Delete(root);

	return 0;
}

int main(int argc, char *argv[])
{
	return create_js();
}

```

## 例程3
创建JSON，添加数组
```c
#include <stdio.h>
#include <stdlib.h>
#include "cJSON.h"

int create_js(void)
{
	int ret = 0;

	cJSON *root = NULL;
	cJSON *js_body = NULL;
	cJSON *js_list1 = NULL;
	cJSON *js_list2 = NULL;
	char *string = NULL;

	root = cJSON_CreateObject();
	if (NULL == root) {
		printf("Error: create root\n");
		ret = -1;
		/*
		 * In fact, cJSON_Delete(NULL) is safe,
		 * so here we can just 
		 * 		goto end;
		 * not need anthor label such as "fail".
		 */
		goto fail;
	}

	js_body = cJSON_CreateArray();
	if (NULL == js_body) {
		printf("Error: create js_body\n");
		ret = -1;
		goto end;
	}

	js_list1 = cJSON_CreateObject();
	if (NULL == js_list1) {
		printf("Error: create js_list1\n");
		ret = -1;
		goto end;
	}
	js_list2 = cJSON_CreateObject();
	if (NULL == js_list2) {
		printf("Error: create js_list2\n");
		ret = -1;
		goto end;
	}

	cJSON_AddItemToObject(root, "body", js_body);

	cJSON_AddItemToArray(js_body, js_list1);
	cJSON_AddItemToArray(js_body, js_list2);

	cJSON_AddStringToObject(js_list1, "name", "aningsk");
	cJSON_AddNumberToObject(js_list1, "status", 100);
	cJSON_AddStringToObject(js_list2, "name", "nikos");
	cJSON_AddNumberToObject(js_list2, "status", 100);

	string = cJSON_Print(root);
	if (NULL != string) {
		printf("%s\n", string);
		free(string);
	}

end:
	/*
	 * Because all others object/array had been added into @root,
	 * we just need to delete @root, and others would also be 
	 * deleted together; not need to delete them manually again.
	 */
	cJSON_Delete(root);
fail:
	return ret;
}

int main(int argc, char *argv[])
{
	return create_js();
}

```

需要说明的是：
1. cJSON\_Delete(NULL)是安全的，所以可以不用像上文代码中多一个“fail”标签。
2. 另外，所有的对象都加入到同一个root节点，在最后释放时，只释放root就可以了。

## 例程4
从JSON中解析信息
```c
#include <stdio.h>
#include <stdlib.h>
#include "cJSON.h"

int main(int argc, char *argv[])
{
	char *out = "{\"name\":\"aningsk\",\"passwd\":\"123456\",\"number\":1}";

	cJSON *json = NULL;
	cJSON *json_name = NULL;
	cJSON *json_passwd = NULL;
	cJSON *json_number = NULL;

	json = cJSON_Parse(out);

//#define CASE_SENSITIVE
#ifdef CASE_SENSITIVE
	json_name = cJSON_GetObjectItemCaseSensitive(json, "name");
	json_passwd = cJSON_GetObjectItemCaseSensitive(json, "passwd");
	json_number = cJSON_GetObjectItemCaseSensitive(json, "number");
#else
	json_name = cJSON_GetObjectItem(json, "Name");
	json_passwd = cJSON_GetObjectItem(json, "Passwd");
	json_number = cJSON_GetObjectItem(json, "Number");
#endif /* CASE_SENSITIVE */

	printf("We get:\n\tname:%s\n\tpasswd:%s\n\tnumber:%d\n",
			json_name->valuestring,
			json_passwd->valuestring,
			json_number->valueint);
	
	cJSON_Delete(json);

	return 0;
}

```

要注意的是：
1. 使用cJSON\_Parse()后，也要记得对应使用cJSON\_Delete()。
2. cJSON\_GetObjectItem()是忽略字母大小写的，同时也有不忽视大小写的。

## 例程5
从JSON中解析信息，以及从JSON中的JSON（一个“list”、一个“other”）解析信息。
```c
#include <stdio.h>
#include <stdlib.h>
#include "cJSON.h"

int main(int argc, char *argv[])
{
	int ret = 0;
	char *info =
		"{\"list\":{\"name\":\"aningsk\",\"age\":10},\"other\":{\"name\":\"nikos\"}}";

	cJSON *root = NULL;
	cJSON *js_list = NULL;
	cJSON *name = NULL;
	cJSON *age = NULL;
	cJSON *js_other = NULL;
	cJSON *js_name = NULL;

	char *string = NULL;

	root = cJSON_Parse(info);
	if (NULL == root) {
		printf("Error: Parse root\n");
		ret = -1;
		goto end;
	}

	string = cJSON_Print(root);
	if (NULL != string) {
		printf("Original info:\n%s\n", string);
		free(string);
	}

	js_list = cJSON_GetObjectItem(root, "list");
	if (NULL == js_list) {
		printf("Error: get list\n");
		ret = -1;
		goto end;
	}
	printf("list type is %d\n", js_list->type);

	name = cJSON_GetObjectItem(js_list, "name");
	if (NULL == name) {
		printf("Error: get name\n");
		ret = -1;
		goto end;
	}
	printf("name type is %d\n", name->type);
	printf("name is %s\n", name->valuestring);

	age = cJSON_GetObjectItem(js_list, "age");
	if (NULL == age) {
		printf("Error: get age\n");
		ret = -1;
		goto end;
	}
	printf("age type is %d\n", age->type);
	printf("age is %d\n", age->valueint);
	printf("age also is %f\n", age->valuedouble);

	js_other = cJSON_GetObjectItem(root, "other");
	if (NULL == js_other) {
		printf("Error: get other\n");
		ret = -1;
		goto end;
	}
	printf("other type is %d\n", js_other->type);

	js_name = cJSON_GetObjectItem(js_other, "name");
	if (NULL == js_name) {
		printf("Error: get name\n");
		ret = -1;
		goto end;
	}
	printf("name type is %d\n", js_name->type);
	printf("name is %s\n", js_name->valuestring);

	printf("\n");
	printf("cJSON_Invalid = %d\n", cJSON_Invalid);
	printf("cJSON_False = %d\n", cJSON_False);
	printf("cJSON_True = %d\n", cJSON_True);
	printf("cJSON_NULL = %d\n", cJSON_NULL);
	printf("cJSON_Number = %d\n", cJSON_Number);
	printf("cJSON_String = %d\n", cJSON_String);
	printf("cJSON_Array = %d\n", cJSON_Array);
	printf("cJSON_Object = %d\n", cJSON_Object);
	printf("cJSON_Raw = %d\n", cJSON_Raw);

end:
	cJSON_Delete(root);
	
	return ret;
}

```

注意的问题：
1. 一般来说，从cJSON对象获得值，要先判断一下type。

## 例程6
从JSON数组中解析信息
```c
#include <stdio.h>
#include <stdlib.h>
#include "cJSON.h"

int main(int argc, char *argv[])
{
	int ret = 0;

	char *string = "{\"list\":[\"name-1\",\"name-2\"]}";
	cJSON *root = NULL;
	cJSON *list = NULL;
	cJSON *item = NULL;
	char *info = NULL;
	int array_size = 0;
	int i = 0;

	root = cJSON_Parse(string);
	if (NULL == root) {
		printf("Error: parse root\n");
		ret = -1;
		goto end;
	}
	
	info = cJSON_Print(root);
	if (NULL == info) {
		printf("Original info:\n%s\n", info);
		free(info);
	}	

	list = cJSON_GetObjectItem(root, "list");
	if (NULL == list) {
		printf("Error: get list\n");
		goto end;
	}

	array_size = cJSON_GetArraySize(list);
	printf("array size is %d\n", array_size);

	for (i = 0; i < array_size; i++) {
		item = cJSON_GetArrayItem(list, i);
		printf("item type is %d\n", item->type);
		printf("item value: %s\n", item->valuestring);
	}

end:
	cJSON_Delete(root);
	return ret;
}

```

好像没什么要太注意的。就是一层一层的解析呗。

嗯嗯，就先写到这里了。

[1]:	https://github.com/DaveGamble/cJSON%20 "cJSON Github地址"
[2]:	https://blog.csdn.net/fengxinlinux/article/details/53121287 "cJSON的使用方法"