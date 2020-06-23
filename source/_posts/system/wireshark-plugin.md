---
title: WireShark 插件开发
type: system
description: 通常我们再项目开发中会定义私有的协议，当通过wireshark查看网络数据包时，如果能够按照我们协议的格式进行显示，并明确显示协议每一位的含义，那将会是很愉快的一件事情，而不需要再去看网络数据晦涩的二进制
date: 2020-06-22 19:10:42
---
# WireShark 插件开发

通常我们再项目开发中会定义私有的协议，当通过wireshark查看网络数据包时，如果能够
按照我们协议的格式进行显示，并明确显示协议每一位的含义，那将会是很愉快的一件事情，
而不需要再去看网络数据晦涩的二进制。


下面举例介绍一下 **WireShark插件** 的开发方法，带你装逼带你飞~~

## 第一步  了解你的协议

想要开发协议插件，首先你要对所开发的协议十分的了解才行。举个例子，下面是我这边的一个私有协议

### 协议格式

![私有portal协议](/images/portal.png)

### 协议字段说明

+ versions : 协议版本，1字节，取值 0 或者 1
+ type : 消息类型，1字节
	+ [1] :  Challenge-Request
	+ [2] :  Challenge-Ack
	+ [3] :  Auth-Request
	+ [4] :  Auth-Ack
	+ [5] :  Logout-Request
	+ [6] :  Logout Ack
	+ [7] :  Auth-AFF-Ack
	+ [8] :  NTF-Logout-Request
	+ [9] :  Ask-Info-Request
	+ [10] : Ask-Info-Ack
	+ [14] : NTF-Logout-Ack
+ authtype : 认证类型，1字节
    + [0] : chap
    + [1] : pap
+ rsv : 保留字节，用于对齐
+ user ip : IP地址，4字节
+ user port : 端口号，2字节
+ errcode : 错误码，1字节
+ attrnum : 后面 TLV格式数据的数量
+ TLV : TLV格式数据
    + T : 类型，1字节
    + L ：长度，1字节，L的值包括T、L本身的长度
    + V ：若干字节

## 第二步  编写插件

了解了协议格式那么就可以开始写自己的插件了，在写之前有几点我觉得是要搞明白的

+ **type** 字段有很多的取值，如何将每个取值的含义显示出来？
+ **errcode** 当 **type** 取值不同时，**errcode** 取值的含义也各不相同，这个该如何显示？
+ **TLV** 数据得长度是变化的，又该如何显示？

## 第三步  使用插件

找到wireshark软件安装目录下的 `init.lua` 文件，在该文件最后添加这样一段，加入你的插件

    dofile("GBportal-v1.0.lua")

添加后，重启wireshark, 那么你的插件就生效了

显示效果，如图：

![插件效果图](/images/portal_view.png)

## 参考资料

[WireShark插件编写之lua库API(1)](https://wiki.wireshark.org/LuaAPI)

[WireShark插件编写之lua库API(2)](https://www.wireshark.org/docs/wsdg_html_chunked/wsluarm_modules.html)

## 附件-插件代码

下面附上我的插件代码，具体细节代码中都有注释，希望对你有参考价值

```lua
-- @brief Portal 协议
-- @author Mr.Piak
-- @date 2019.09.11
----------------------------------------------
--[[
			portal协议报文格式
			
---------------------------------------------------------
|========8|========16|========24|========32|
|version  |  type    | authtype | rsv      |
|            userIP                        |
|  userport          |  errcode | attrnum  |
|   T     |   L      |          ...        |	TLV数据 T:1byte  L: 1byte
|           ...V...                        |
|             .....                        |
|========8|========16|========24|========32|
---------------------------------------------------------
]]
-- create a new dissector
local NAME = "GB-Portal"    -- 定义一个变量，用来保存协议名称，供后面使用
local PORT = 2000           -- 协议的UDP端口号
local portal = Proto(NAME, "GB-Portal Protocol")    -- 函数 Proto 用来创建一个协议，返回一个协议对象 protal

--=============================================================================
-- 定义 报文头部分  msgheader
--=============================================================================
-- 定义一个协议头变量，这个变量的目的是，在显示的时候将协议的每个字段都作为它的子成员，包含显示
local pf_portalheader        	= ProtoField.none("portal.msgheader", "Portal Header")

--[[ /*字段定义 方法1*/
local pf_version        		= ProtoField.uint8("portal.version", "Version")
local pf_type         			= ProtoField.uint8("portal.type", "Type")
local pf_authtype        		= ProtoField.uint8("portal.authtype", "Authtype")
local pf_rsv         			= ProtoField.uint8("portal.rsv", "rsv")
local pf_serialID        		= ProtoField.uint16("portal.serialID", "Serial id")
local pf_reqID         			= ProtoField.uint16("portal.reqID", "Req id")
local pf_userIP         		= ProtoField.uint32("portal.userIP", "User ip")
local pf_userport        		= ProtoField.uint16("portal.userport", "User port")
local pf_errcode         		= ProtoField.uint8("portal.errcode", "Err code")
local pf_attrnum         		= ProtoField.uint8("portal.attrnum", "Attr num")
]]

-- 字段定义 方法2
-- 使用 ProtoField.new（）  ，根据协议内容为每个字段创建一个字段对象
-- UINT8  表示 占 1 个字节
-- UINT16 表示 占 2 个字节
-- IPV4   表示这是一个IP地址
-- 更多类型可以查看 wireshark lua API ， 参考链接中有API的网址
local pf_type         			= ProtoField.new("Type","portal.type",ftypes.UINT8)
local pf_version        		= ProtoField.new("Version","portal.version",ftypes.UINT8)
local pf_authtype        		= ProtoField.new("Authtype","portal.authtype",ftypes.UINT8)
local pf_rsv         			= ProtoField.new("Rsv","portal.rsv",ftypes.UINT8)
local pf_userIP         		= ProtoField.new("Userip","portal.userIP",ftypes.IPv4)
local pf_userport        		= ProtoField.new("User port","portal.userport",ftypes.UINT16)
local pf_errcode         		= ProtoField.new("Err code","portal.errcode",ftypes.UINT8)
local pf_attrnum         		= ProtoField.new("Attr num","portal.attrnum",ftypes.UINT8)

-- attrs 用来囊括 所有的TLV格式数据
local attrs 	= ProtoField.none("gbcomcapwap.tlv", "Attrs")
-- tlv 用来囊括单个TLV格式数据
local tlv 		= ProtoField.none("gbcomcapwap.tlv", "attr")
-- TLV 格式数据的 T 字段
local tlv_type 	= ProtoField.uint8("gbcomcapwap.tlv_type", "type")
-- TLV 格式数据的 L 字段
local tlv_len 	= ProtoField.uint8("gbcomcapwap.tlv_len", "length")
-- TLV 格式数据的 V 字段
local tlv_val 	= ProtoField.string("gbcomcapwap.tlv_val", "value")

--=============================================================================
-- 为协议添加字段
-- 注意这个时候只是 字段对象 和 协议对象 产生关联 , 大括号内的字段顺序是随意的，目前还
-- 没有开始解析协议 
--=============================================================================
portal.fields = {pf_portalheader,pf_version,pf_type,pf_authtype,pf_rsv,
                pf_serialID,pf_reqID,pf_userIP,pf_userport,
                pf_errcode,pf_attrnum,attrs,tlv,tlv_type,tlv_len,tlv_val}


-- dissect packet
-- 定义数组， type字段取值 和含义
local portalTypes = {
	[1] = "  Challenge-Request",
	[2] = "  Challenge-Ack",
	[3] = "  Auth-Request",
	[4] = "  Auth-Ack",
	[5] = "  Logout-Request",
	[6] = "  Logout Ack",
	[7] = "  Auth-AFF-Ack",
	[8] = "  NTF-Logout-Request",
	[9] = "  Ask-Info-Request",
	[10] = "  Ask-Info-Ack",
	[14] = "  NTF-Logout-Ack"
}

-- 定义当type=2时，errorcode含义
local ChallenageErrorCode = {
	[0] = "  Challenge-Success",
	[1] = "  Challenge-Refused",
	[2] = "  Challenge-Connection-Exist",
	[3] = "  Challenge-Try-Later",
	[4] = "  Challenge-Fail",
}

-- 定义当type=4时，errorcode含义
local AuthAckErrorCode = {
	[0] = "  Auth-Success",
	[1] = "  Auth-Refused",
	[2] = "  Auth-Connection-Exist",
	[3] = "  Auth-Try-Later",
	[4] = "  Auth-Fail",
}

-- 定义当type=5时，errorcode含义
local LogoutReqErrorCode = {
	[0] = "  Logout-Request",
	[1] = "  Logout-Request-Timeout",
}

-- 定义当type=6时，errorcode含义
local LogoutAckErrorCode = {
	[0] = "  Logout-Ack-Sucess",
	[1] = "  Logout-Ack-Refused",
	[2] = "  Logout-Ack-Fail"
}

-- TLV 格式数据  不同  T 的含义
local attrType = {
	[1] = "  UserName",
	[2] = "  PassWord",
	[3] = "  Challenage",
	[4] = "  ChapPassword"
}
local portalHdr_len = 12		--消息头长度,不包括TLV数据长度

-- 重载 portal对象的 dissector 方法，用来解析协议
function portal.dissector (tvb, pinfo, tree)
    -- set the protocol column to show our protocol name
    pinfo.cols.protocol:set(NAME)                           -- wireshark 报文列表中显示的报文协议名称
	local pktlen = tvb:reported_length_remaining()          -- 报文总长度

    -- 正式开始解析
	local portal_tree = tree:add(pf_portalheader,      tvb:range(0,portalHdr_len))  -- 首先加载 pf_portalheader对象，返回 portal_tree
    local offset=0                                          -- 定义变量，用来记录报文偏移量
    portal_tree:add(pf_version,     tvb:range(0,1))         -- 添加 version 字段 占 1 字节,range 从0 开始
    offset = offset +1                                      -- 
    local portal_type_tree = portal_tree:add(pf_type,     tvb:range(offset,1)) -- 这里主要是为了 保存 type字段的 tree对象
	local portal_type = tvb:range(offset,1):uint()          -- 获取type字段的值
	portal_type_tree:append_text(portalTypes[portal_type])  -- 把 type的值含义字段，追加显示
    offset = offset +1
    portal_tree:add(pf_authtype,      tvb:range(offset,1))  -- 后面的同上
    offset = offset +1
    portal_tree:add(pf_rsv,      tvb:range(offset,1))
    offset = offset +1
    portal_tree:add(pf_serialID,   tvb:range(offset,2))
    offset = offset +2
    portal_tree:add(pf_reqID,   tvb:range(offset,2))
    offset = offset +2
    portal_tree:add(pf_userIP,    tvb:range(offset,4))
    offset = offset +4
    portal_tree:add(pf_userport,    tvb:range(offset,2))
    offset = offset +2
    local portal_errcode_tree = portal_tree:add(pf_errcode,        tvb:range(offset,1))
	local error_code = tvb:range(offset,1):uint()
    offset = offset +1
    portal_tree:add(pf_attrnum,       tvb:range(offset,1))
	local attrnums = tvb:range(offset,1):uint()         -- 一直解析到 attrnum字段
    offset = offset +1
	
	
	-- 根据不同type设置 不同error code的含义  
	if ( portal_type == 2 ) then
		portal_errcode_tree:append_text(ChallenageErrorCode[error_code])
	elseif ( portal_type == 4 ) then
			portal_errcode_tree:append_text(AuthAckErrorCode[error_code])
	elseif ( portal_type == 5 ) then
			portal_errcode_tree:append_text(LogoutReqErrorCode[error_code])
	elseif ( portal_type == 6 ) then
			portal_errcode_tree:append_text(LogoutAckErrorCode[error_code])
	end

	
	
	local value_len = 0
	local attrs_tree = portal_tree:add(attrs,      tvb:range(offset,pktlen - portalHdr_len))
    attrs_tree:append_text(" ("..tostring(attrnums)..")")   -- 将TLV格式数据的总数目 追加显示
    -- 依据TLV 格式数据数目，进行for循环，解析每个 TLV 数据
	for i = 1,attrnums do
		local tlv_tree = attrs_tree:add(tlv,      tvb:range(offset,2))
		tlv_tree:append_text("("..tostring(i)..")")					-- 设置第N个属性
		local tlv_type_tree = tlv_tree:add(tlv_type,tvb:range(offset,1))
		local tlv_type_v = tvb:range(offset,1):uint() 		-- 获取type的值
		tlv_type_tree:append_text(attrType[tlv_type_v])		-- 设置type的含义
		offset = offset +1
		tlv_tree:add(tlv_len,tvb:range(offset,1))
		value_len = tvb:range(offset,1):uint() - 2
		offset = offset +1
		tlv_tree:add(tlv_val,tvb:range(offset,value_len))
		offset = offset + value_len
	end
	
    pinfo.cols.info = portalTypes[portal_type]  -- 在 wireshark 报文列表的 列栏中显示 每个报文 type 含义

end

-- register this dissector
DissectorTable.get("udp.port"):add(PORT, portal)
```