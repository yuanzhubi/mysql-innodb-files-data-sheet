# mysql innodb引擎文件datasheet

引言：
  本文将主要介绍mysql innodb引擎下的各种文件的存储格式，并尽力覆盖和追踪一些在旧版本和新版本中的变化。

第一章：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6">.frm类型文件</a>

    第一节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%B8%80%E8%8A%82frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%BB%93%E6%9E%84">.frm类型文件的基本结构</a>
    第二节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%BA%8C%E8%8A%82macros-frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6%E4%B8%AD%E4%BD%BF%E7%94%A8%E5%88%B0%E7%9A%84%E4%B8%80%E4%BA%9B%E5%AE%8F%E5%B8%B8%E9%87%8F%E5%AE%9A%E4%B9%89">.frm类型文件中使用到的一些宏常量定义</a>
    第三节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%B8%89%E8%8A%82%E6%96%87%E4%BB%B6%E5%A4%B4%E9%83%A8%E5%88%86">文件头部分</a>
    第四节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E5%9B%9B%E8%8A%82%E8%A1%A8%E7%9A%84%E6%89%80%E6%9C%89%E7%B4%A2%E5%BC%95%E7%9A%84%E4%BF%A1%E6%81%AF">表的所有索引的信息</a>
    第五节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%BA%94%E8%8A%82%E9%BB%98%E8%AE%A4%E5%80%BC%E4%BF%A1%E6%81%AF">默认值信息</a>
    第六节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E5%85%AD%E8%8A%82%E8%A1%A8%E7%9A%84%E6%89%A9%E5%B1%95%E4%BF%A1%E6%81%AF">表的扩展信息</a>
    第七节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%B8%83%E8%8A%82-%E8%A1%A8%E7%9A%84%E8%A1%A8%E5%8D%95%E4%BF%A1%E6%81%AFforminfo">表的表单信息（forminfo）</a>
    第八节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E5%85%AB%E8%8A%82%E8%A1%A8%E7%9A%84screen%E4%BF%A1%E6%81%AFscreen_info">表的screen信息(screen_info)</a>
    第九节：<a href="https://github.com/yuanzhubi/mysql-innodb-files-data-sheet/wiki/.frm%E7%B1%BB%E5%9E%8B%E6%96%87%E4%BB%B6#%E7%AC%AC%E4%B9%9D%E8%8A%82%E8%A1%A8%E7%9A%84%E6%89%80%E6%9C%89%E5%AD%97%E6%AE%B5%E4%BF%A1%E6%81%AF">表的所有字段信息</a>
    
