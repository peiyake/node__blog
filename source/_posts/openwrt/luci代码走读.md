


### 修改接口协议后，默认uci配置字段配置

1. 文件 **qca\feeds\luci\modules\luci-mod-admin-full\luasrc\model\cbi\admin_network\ifaces.lua**
2. 默认保留的字段，以及如何添加固定字段 

```lua
		-- clear options  这里显示默认保留的字段
		local k, v
		for k, v in pairs(m:get(net:name())) do
			if k:sub(1,1) ~= "." and
			   k ~= "type" and
			   k ~= "ifname" and
			   k ~= "_orig_ifname" and
			   k ~= "_orig_bridge"
			then
				m:del(net:name(), k)
			end
		end

		-- set proto
		m:set(net:name(), "force_link",'1')     -- 添加固定字段
		m:set(net:name(), "proto", proto:proto())
		m.uci:save("network")
		m.uci:save("wireless"
```

### uci network默认配置 

1. 默认配置修改这个脚本 **package\base-files\files\lib\functions\uci-defaults.sh**
2. 网口对应关系配置,修改这个脚本 **target\linux\ipq\base-files\etc\uci-defaults\network** (这个文件根据不同硬件平台对应的文件不一样)


## luci 添加自定义协议

1. 协议文件自定义目录 `qca\feeds\luci\protocols`
2. 仿照已存在的协议，定义自己的协议文件
3. 写好后，更新feeds，然后`make menuconfig` 找找luci->proto  将自定义的协议选中后才会被编译