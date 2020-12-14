

## 问题描述

```
Create index file './feeds/gbcom.index' 
Collecting package info: done
Updating feed 'eogre' from '../gbcom/feeds/netd/eogre' ...
Create index file './feeds/eogre.index' 
grep: feeds/eogre/Makefile:$(eval: No such file or directory
grep: $(call: No such file or directory
grep: BuildPackage,eogre))/Makefile: No such file or directory
/home/peiyake/gitlab/wifi/QSDK_SPF11.1.CSU2_IPQ6018/feeds/eogre.tmp/info/.files-packageinfo.mk:1: *** target pattern contains no `%'.  Stop.
```

原因是 `eogre` 这个feed冲突了，把这个改成其它名字就可以了。