# mysql innodb引擎文件datasheet

引言：
本文将主要介绍mysql innodb引擎下的各种文件的存储格式，并尽力覆盖和追踪一些在旧版本和新版本中的变化。

第一章：.frm类型文件

导语：

    mysql持久化数据存储中最常见的细粒度单元即是：表。一个表的逻辑结构在create语句和alter语句中被定义和修改定义，并且和实际的物理存储引擎无关（无论是用myisam或者inno，你都可以用几乎一致的create语句而唯一的区别在于存储引擎字段的值不同。）。对于表table1, 这个结构信息在过去的mysql版本中存储于table1.frm。 .frm文件作为mysql表逻辑结构的持久化存储文件，在官方网站上并不能找到清晰的和完整的介绍文档(https://dev.mysql.com/doc/internals/en/frm-file-format.html 这个网页的内容并不完整而且还有一些谬误)，很难想象如此重要的文件结构mysql基本靠“源码就是文档”这种模式活到了今天！！事实上笔者在对mysql源码分析中也感觉frm文件格式内部组织非常之混乱(等一下你就可以看到他是如何反人类的区分表和视图的)。还好mysql现在主线的维护人员也感觉这样没法维系下去了，最新版本的mysql已经不再单独创建frm文件。

   笔者下载了最新的mysql来创建表已经看不到frm文件， git上的源码也不再有create_frm这个函数而只有read_frm_file这个函数来支持从旧版本数据库的表导入到新版本的库中，具体是从哪一个版本起frm文件消失已经不可考证(https://dev.mysql.com/doc/refman/8.0/en/data-dictionary-file-removal.html 从源码分析上应该早于8.0,从一些介绍中则可以看到在5.7应该还存在https://dev.mysql.com/doc/refman/5.7/en/innodb-file-format-compatibility.html )。 不过目前为止，绝大多数linux操作系统上自带的mysql版本依然会在建表时会创建.frm信息，这些版本的mysql数据库可以相信仍然会伴随工业界相当长时间，分析.frm文件格式依然是很有意义的一件事情。而且.frm文件不再使用也标志他不会有新的逻辑被加入，不会有新的历史包袱要兼容，一切都被画上了句号。

综上，所以本章即从.frm文件介绍如何从他的内部数据中去分析出一个表的逻辑结构。注意.frm文件并不能为你描述触发器等内容，只能从中得到一个静态的逻辑结构。

作为当前已经不会创建frm的版本，https://github.com/mysql/mysql-server/blob/8.0/sql/table.cc 中的read_frm_file 函数可以帮我们确认哪些字段即使是当前mysql导入旧数据时也不会考虑的，哪些字段虽然并不使用但是会作为数据合法性标识符，哪些字段会因为创建时的不同数据库版本而存在不同或者变化的含义。 而我们也会从https://github.com/mysql/mysql-server/blob/2c1aab1fe5a17e8804124be22e90f5a598cf7305/sql/table.cc 这个历史版本中的create_frm函数和unireg.cc中的mysql_create_frm函数中分析那些现在很少使用的字段是怎么写入进去的。

注意：

    在frm文件中存储的多字节整数使用小端存储，在本文中出现0001，则表示这是一个2字节整数，低地址字节是00高地址字节是01，所以这个数字是256。如果不特别指出，默认所有整数是无符号类型整数。
    
    在frm文件中存储的字符串并不一定使用00结尾，也可能使用ff结尾。如果不特别指出，默认所有字符串内容没有任何结尾符号。
    
    test(x) 表示如果x为真则test(x)返回1否则返回0
    
第一节：.frm类型文件的基本结构

    .frm文件内部数据大致分成5部分，

    1.文件头部分。主要存储一些诸如版本，创建时引擎的总体信息和其他信息的长度或位置等信息。
    2.表的所有索引的信息。
    3.默认值信息。
    4.附加信息，诸如表分区，表注释之类的信息。
    5.表的所有字段的信息。
    
第二节：.frm类型文件中使用到的一些宏常量定义

   FRM_VER 定义在https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/include/mysql_version.h  目前是6
   
   IO_SIZE 定义在https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/include/my_io.h 能够看到的版本都是4096
   
   MAX_REF_PARTS 定义在https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/sql/sql_const.h  目前是16，表示联合索引中字段最大数量
  
   NAME_LEN 定义在https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/sql/sql_const.h 目前是NAME_CHAR_LEN * SYSTEM_CHARSET_MBMAXLEN = 64 * 3 = 192
   
   HA_OPTION_LONG_BLOB_PTR 定义在https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/include/my_base.h 目前是8
   
   
第三节：文件头部分    

   文件头部分从第0字节开始，而长度则由第6和第7字节组成的二字节整数确定。
   
    文件头信息,考虑到理解习惯,位置我们使用“0x”表示法
   
   | 位置  | 长度  | 解释 |
   | :--- | :---  |:---- |
   | 0x0000 | 2 | 固定写死fe01。(如果是.frm关联的是一个视图，那么前9个字节是TYPE=VIEW的ascii码，参见https://github.com/mysql/mysql-server/blob/b93c1661d689c8b7decc7563ba15f6ed140a4eb6/sql/table.cc#L7435) |
   | 0x0002 | 1     | FRM_VER + 3 + test(表是否有varchar)|
   | 0x0003 | 1     | 数据库类型枚举，参见定义在sql/handler.h的enum legacy_db_type|
   | 0x0004 | 2     | 固定写死03. forminfo_length_offset, 表项信息相对64字节的偏移量|
   | 0x0006 | 2     | IO_SIZE宏，能够看到的版本都是4096|
   | 0x0008 | 2     | 固定写死0100|
   | 0x000a | 4     | 一个计算很复杂的信息，没有看到任何一个版本在解析时使用了这个字段 <br>参见https://github.com/mysql/mysql-server/blob/2c1aab1fe5a17e8804124be22e90f5a598cf7305/sql/table.cc#L3862 可阅读他的计算方法，应该视作整个文件的最终大小|
   | 0x000e | 2     | 索引信息部分最大存储长度，计算方法如下<br>key_length = keys * (8 + MAX_REF_PARTS * 9 + NAME_LEN + 1) + 16 + key_comment_total_bytes;<br>keys表示有多少个索引<br>key_comment_total_bytes表示有所有索引注释长度加2后的总和<br>如果key_length小于ffff，那么在当前字段存储key_length；否则key_length到0x002f存储，当前字段填ffff|
   | 0x0010 | 2     | 默认值信息部分开头的位图字节长度rec_length|
   | 0x0012 | 4     | 表最大行数|
   | 0x0016 | 4     | 表最小行数|  
   | 0x001a | 1     | test((表最大行数 == 1) &&  (表最小行数 == 1) && (索引数量 == 0))|
   | 0x001b | 1     | 只支持按02来解析|
   | 0x001c | 2     | 索引内容部分信息长度|
   | 0x001e | 2     | HA_OPTION_LONG_BLOB_PTR|
   | 0x0020 | 2     | 主要填0005（mysql5）。如果是0000，那么版本小于3.23，接下来的参数都没用了 |
   | 0x0022 | 4     | 表的平均单行长度，可以用来指示分表或辅助展示|
   | 0x0026 | 1     | 表的默认字符集选项id, 这个id可用值可以从SELECT ID, Collation, Charset FROM INFORMATION_SCHEMA.COLLATIONS 查询 |
   | 0x0027 | 1     | 注释显示计划用作TRANSACTIONAL and PAGE_CHECKSUM clauses of CREATE TABLE，但未实际使用，填0  |
   | 0x0028 | 1     | 行类型ROW_FORMAT参数。<br>参见https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/sql/handler.h 中的row_type选项 |
   | 0x0029 | 1     | 可以作为高位和0x0026组合成一个2字节整数来获取完整的编码和collation信息。<br>参见https://github.com/mysql/mysql-server/blob/b93c1661d689c8b7decc7563ba15f6ed140a4eb6/sql/table.cc#L1435    |
   | 0x002a | 1     |  number of pages to sample during stats estimation, if used, otherwise 0. |
   | 0x002b | 1     |  写死0 |
   | 0x002c | 1     |  Automatic recalc of stats, 类型为enum_stats_auto_recalc |
   | 0x002d | 2     |  写死0 |
   | 0x002f | 4     |  索引信息部分最大存储长度key_length的4字节存储形式 |
   | 0x0033 | 4     |  mysql版本号的小端存储，MYSQL_VERSION_ID from include/mysql_version.h <br>	c0c30000=>50112=>5.01.12 |
   | 0x0037 | 4     |  附加信息长度 |
   | 0x003b | 2     |  extra_rec_buf_length, 并未被实际使用的一个字段。多数填0 |
   | 0x003d | 1     |  db默认分区类型，如果非0则表启用了分区表 |
   | 0x003e | 2     |  KEY_BLOCK_SIZE参数 |
   0x0040-0x1000
   
   
第四节：表的所有索引的信息  

    表的所有索引的信息部分从第IO_SIZE(只见过4096)字节开始，而长度则由第0x0e字节处的2字节无符号整数决定如果他不等于0xffff，否则是0x2f处的4字节整数。设cursor游标是一个指向所在行位置的uint8指针，new_frm_ver=第2字节数-FRM_VER
    
   | 位置  | 长度  | 解释 |
   | :--- | :---  |:---- |
   | 0x1000 | 4 | key_count(有多少个索引), key_parts（索引们的所有字段数量的和）<br>if(cursor[0]<127)key_count = cursor[0], key_parts = cursor[1], cursor[3]=cursor[4]=0<br>else key_count = cursor[0]-128+(cursor[1]%2) * 128, key_parts=((uint16*)cursor)[1]|
   | 0x1004 | 2 | key_extra_info（扩展信息长度，是本节最后的索引名称和索引注释2个部分数据的总长度）|
   | 0x1006 | 数组变长 | 索引内容部分信息。每一个数组元素中包含索引内容信息（高版本8字节低版本4字节），接着是这个索引的每一个字段内容信息（高版本9字节低版本7字节），详细代码展开见本节最后|
   |  |变长| 开头一个ff, 然后是每一个索引的名称，用ff结尾。结束后是一个00。<br>注意即使一个索引也没有，开头的ff和结尾的00都不能省略。|
   |  |变长| 每一个索引的2字节注释长度和注释内容，没有注释也需要写一个2字节的0。|
   
   注意https://dev.mysql.com/doc/refman/5.6/en/index-extensions.html 所支持的索引自动主键扩展，实测并不会算到索引的字段数量里去。
   
   索引内容信息
   
   | 名字  | 长度  | 解释 |
   | :--- | :---  |:---- |
   | flags | 2 | 注意可以参与https://github.com/mysql/mysql-server/blob/b93c1661d689c8b7decc7563ba15f6ed140a4eb6/include/my_base.h#L459 这里的位运算，低版本只有1字节|
   | key_length | 2     |  索引长度|
   | user_defined_key_parts | 1     |  索引字段个数|
   | algorithm | 1     |  索引算法，老版本是固定的|
   | block_size | 1     |  建索引的KEY_BLOCK_SIZE，老版本没有|
   |  | 1     |  未使用|
   
   ```c++
    if (new_frm_ver >= 3) {
      keyinfo->flags = (uint)uint2korr(strpos) ^ 1;
      keyinfo->key_length = (uint)uint2korr(strpos + 2);
      keyinfo->user_defined_key_parts = (uint)strpos[4];
      keyinfo->algorithm = (enum ha_key_alg)strpos[5];
      keyinfo->block_size = uint2korr(strpos + 6);
      strpos += 8;
    } else {
      keyinfo->flags = ((uint)strpos[0]) ^ 1;
      keyinfo->key_length = (uint)uint2korr(strpos + 1);
      keyinfo->user_defined_key_parts = (uint)strpos[3];
      // The algorithm was HA_KEY_ALG_UNDEF in 5.7
      keyinfo->algorithm = HA_KEY_ALG_SE_SPECIFIC;
      strpos += 4;
    }
   ```
   
   索引的字段内容信息
   
   | 名字  | 长度  | 解释 |
   | :--- | :---  |:---- |
   | fieldnr | 2 | Fieldnum in UNIREG|
   | offset | 2     |  offset in record plus 1|  
   | key_part_flag | 1     |   0 or HA_REVERSE_SORT，低版本这个字段低7位为索引数据长度，大于127则解析为HA_REVERSE_SORT|
   | key_type | 2     |  enum ha_base_keytype|
   | length | 2     |  索引数据长度，所以高版本支持65535长的索引长度而低版本仅支持到127|
    
    ```c++
      key_part->fieldnr=	(uint16) (uint2korr(strpos) & FIELD_NR_MASK);
      key_part->offset= (uint) uint2korr(strpos+2)-1;
      key_part->key_type=	(uint) uint2korr(strpos+5);
      // key_part->field=	(Field*) 0;	// Will be fixed later
      if (new_frm_ver >= 1){
	     key_part->key_part_flag= *(strpos+4);
	     key_part->length=	(uint) uint2korr(strpos+7);
	     strpos+=9;
      }else{
	     key_part->length=	*(strpos+4);
	     key_part->key_part_flag=0;
	     if (key_part->length > 128){
	         key_part->length&=127;		/* purecov: inspected */
	         key_part->key_part_flag=HA_REVERSE_SORT; /* purecov: inspected */
	      }
	     strpos+=7;
      }
    ```
   第四节：默认值信息
   	
	注意这个部分应该是从索引字段的开始位置偏移key_length（定义在文件头47字节）后才开始, 此外这个部分长度是在文件头中10处标明。
	
	| 名字  | 长度  | 解释 |
       	| :--- | :---  |:---- |
       	| null_bitmap | 变长 |如果没有默认值为null的字段，那么长度为1，第一字节为0。<br>否则第一字节最低bit为1，位图长度为(null_column_count + 7 ) / 8 字节<br>从低位到高位，从低字节到高字节标识某个列的默认值是否为null（是null就把比特位置1，注意某列如果是位域类型，则还需要跳过bitsize&7位），最后一个字节多余比特位用1填充。|
	| 默认值字段 | 变长数组 |非null默认值根据自己的不同类型根据自己的存储方法存储，字符串类型等使用表的默认编码类型进行编码。<br>可阅读https://github.com/mysql/mysql-server/blob/b93c1661d689c8b7decc7563ba15f6ed140a4eb6/sql/field.cc 中不同类型作为Field抽象类的子类去实现各种store方法|
	
   第五节：表的扩展信息
      
       以下字段都是可选的，当发现已解析字段长度超过了文件头里的“附加信息长度”，则剩余字段不存在。
       
       | 名字  | 长度  | 解释 |
       | :--- | :---  |:---- |
       | connect_string.length | 2 |连接字符串长度|
       | connect_string |  connect_string.length   |  连接字符串|  
       | str_db_type.length | 2     |数据库引擎名字长度|
       | str_db_type| str_db_type.length     |数据库引擎名字|
       | partition_info_str | 4     |分区信息长度|
       | partition_info_str | partition_info_str.len     |分区信息|
       | 00|1|字符串结尾信息|
       | auto_partitioned|1|是否自动分区|
       | 索引解析器名字数组 | 数组变长     |每一个索引的解析器名字,00结尾|
       | table_comment.length | 2 |表的注释信息长度. <br>注意table_comment.length和table_comment需要forminfo第46个字节是255才存在|
       | table_comment |  table_comment.length   | 表的注释长度|
       | table_format_length|2|5.1.20才有的表format信息开始。<br>这里这个长度包含了从&table_format_length开始到column properties的所有字段结束的数据总长度|
       | table_format_flags|4|format标记|
       | 0000|2|未使用|
       | tablespace and 00 |字符串变长| 用00结尾的tablespace|
       | column properties|数组变长|field_storage_type+column_format<<COLUMN_FORMAT_SHIFT组成的数组<br>表format信息到这里结束|
       | compress.length | 2 |压缩算法字符串长度|
       | compress |  compress.length   |  压缩算法字符串内容|  
       | encrypt.length | 2 |加密算法字符串长度|
       | encrypt |  encrypt.length   |  加密算法字符串内容| 
       
       
   第六节: 表的表单信息（forminfo） 
  
   	这部分信息主要用来辅助。这个部分的位置，需要从文件的第64字节偏移forminfo_length_offset个字节再读取4个字节小端整数pos=*(uint32*)(file+64+forminfo_length_offset), 注意forminfo_length_offset定义在文件中的第4字节。这个部分的大小固定为288字节。以下位置从forminfo开始算起,有很多没啥大用的字段这里就不做更多的解释，如有疑问可直接参阅https://github.com/mysql/mysql-server/blob/2c1aab1fe5a17e8804124be22e90f5a598cf7305/sql/unireg.cc#L860
	
	|相对forminfo的位置  | 长度  | 解释 |
   	| :--- | :---  |:---- |
   	| 0x0100 | 1 |  (字段数量-1)/19+1; 参见https://github.com/mysql/mysql-server/blob/2c1aab1fe5a17e8804124be22e90f5a598cf7305/sql/unireg.cc#L552|
   	| 0x0102 | 2     | 表字段数量|
	| 0x0104 | 2     | 表的screen信息长度|
	| 0x010a | 2     | 默认值信息部分开头的位图字节长度rec_length|
   	| 0x010e | 2     | 字段内部数量|
   	| 0x0110 | 2     | 字段内部子部分数量|
   	| 0x0112 | 2     | 字段内部总长度| 
	| 0x0114 | 2     | 时间戳字段位置| 
	| 0x011a | 2     | null字段个数| 
	| 0x011c | 2     | 列的评论总字节数| 
	
	
  第七节：表的screen信息(screen_info)
  	 
	 用来展示用的，解析的时候就麻烦用forminfo里的第0x0104的信息来跳过吧。有兴趣就请阅读https://github.com/mysql/mysql-server/blob/2c1aab1fe5a17e8804124be22e90f5a598cf7305/sql/unireg.cc#L537
	
  第八节：表的列字段信息
  
  	表的每个字段固定占17字节。
	|相对每个字段开始的偏移  | 长度  | 解释 |
   	| :--- | :---  |:---- |
   	| 02 | 1 | 整形字段大小|
	| 03 | 2 | 字段大小，按照实际编码来设定。例如varchar(4), gbk这里就存2*4，utf8这里就存3*4|
	| 05 | 3 | 默认值内部存储偏移位置，从1起序，注意这是个3字节无符号整数|
   	| 08 | 2 | 表字段数量|
	| 0a | 1 | FRM_context::utype 参见https://github.com/mysql/mysql-server/blob/b93c1661d689c8b7decc7563ba15f6ed140a4eb6/sql/table.h#L3754|
	| 0b | 1 | chh, 编码类型id高位，字节|
	| 0c | 1 | interval_nr 类型修正偏移值|
	| 0d | 1 | 字段类型，参见https://github.com/mysql/mysql-server/blob/4f1d7cf5fcb11a3f84cff27e37100d7295e7d5ca/libbinlogevents/export/binary_log_types.h#L52|
	| 0e | 1 | 如果字段是地理类型，那么这个字节表示地理类型<br>否则chl,编码类型id低位字节，chl+chh<<8表示编码id|
	| 0f | 2 | 注释长度|
	
  	 
	 
	
   	
   
