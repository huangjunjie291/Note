首先redis是一个KEY-VALUE存储方式
KEY是String，而VALUE有五大数据类型：String,List,Hash,Set,ZSet
而这五大数据类型在redis中的底层实现包括：SDS，双向链表，哈希表，压缩列表，listpack，quicklist,整数集合，跳表。