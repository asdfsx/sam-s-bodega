+++
title = "odoo自定义模块"
type = "post"
author = "asdfsx"
draft = false
date = "2017-01-13T11:46:33+08:00"
tags = [
  "python",
  "odoo",
]
topics = [
  "topic 1",
]
keywords = [
  "python",
  "odoo",
]
description = "odoo自定义模块"

+++

复制 http://blog.sunansheng.com/python/odoo/odoo.html 中的请假单例子，创建一个带工作流的例子，工作环境是`odoo 10.0`。  
PS：在这个版本中官方自带的请假模块删掉了工作流～～何等的卧槽！

### 创建模块模版：  

```
python odoo/odoo-bin scaffold qingjia odoo_dev/
```

在`odoo_dev`目录中可以找到新创建的模块，进入目录以后可以看到如下的目录结构

```
controllers  demo  __init__.py  __manifest__.py  models  security  views
```

`__init__.py`不需要修改  
`__manifest__.py`需要增加一点东西

```
"application": True,
```

### 创建模型
修改`models/model.py`文件。添加新的模型

```
# -*- coding: utf-8 -*-

from odoo import models, fields, api

class Qingjd(models.Model):
    _name = 'qingjia.qingjd'
    
    name = fields.Many2one('res.users', string="申请人", required=True)
    days = fields.Float(string="天数", required=True)
    startdate = fields.Date(string="开始日期", required=True)
    reason = fields.Text(string="请假事由")
    
    def send_qingjd(self):
        self.sended = True
        return self.sended
        
    def confirm_qingjd(self):
        self.state = 'confirmed'
        return self.state
```

### 修改controller
修改`controllers/controller.py`文件。由模版创建的`controller`已经提供了最基础的功能，只需要去掉注释，做简单的修改就可以使用（主要是和新添加的模型做个对应）。

```
# -*- coding: utf-8 -*-
from odoo import http
class Qingjia(http.Controller):
    @http.route('/qingjia/qingjia', auth='public')
    def index(self, **kw):
        return "Hello, world"
        
    @http.route('/qingjia/qingjia/objects/', auth='public')
    def list(self, **kw):
        return http.request.render('qingjia.listing', {
            'root': '/qingjia/qingjia',
            'objects': http.request.env['qingjia.qingjd'].search([]),
        })

    @http.route('/qingjia/qingjia/objects/<model("qingjia.qingjd"):obj>/', auth='public')
    def object(self, **kw):
        return http.request.render('qingjia.object', {
            'object': obj,
        })

```

### 修改模版
在`controller`中，可以看到使用了两个模版`qingjia.listing`和`qingjia.object`。这两个模版定义在`views/templates.xml`文件中，只不过现在被注释掉了。只需要去掉注释就可以了，不需要做任何修改。

```
<odoo>
    <data>
        <template id="listing">
            <ul>
                <li t-foreach="objects" t-as="object">
                    <a t-attf-href="#{ root }/objects/#{ object.id }">
                        <t t-esc="object.display_name"/>
                    </a>
                </li>
            </ul>
        </template>
        <template id="object">
            <h1><t t-esc="object.display_name"/></h1>
            <dl>
                <t t-foreach="object._fields" t-as="field">
                    <dt><t t-esc="field"/></dt>
                    <dd><t t-esc="object[field]/></dd>
                </t>
            </dl>
        </template>
    </data>
</odoo>
```

### 修改视图
视图的信息定义在`views/views.xml`中。模版生成内容都被注释了，直接添加自己的内容好了。我们也可以从零开始。

首先先把文件的框架建好
```
<odoo>
    <data>
    </data>
</odoo>
```

##### 修改导航按钮
```
<menuitem id="menu_qingjia" name="请假" sequence="0"></menuitem>
<menuitem id="menu_qingjia_qingjiadan" name="请假单" parent="menu_qingjia"></menuitem>
<menuitem id="menu_qingjia_qingjiadan_qingjiadan" name="列表" parent="menu_qingjia_qingjiadan" action="action_qingjia_qingjd"></menuitem>

```

##### 修改列表页
在`views/views.xml`中增加对列表页的定义。可以借鉴模版生成的`qingjia.list`模版，只需要修改下字段就可以了。

```
<record model="ir.ui.view" id="qingjia.list">
    <field name="name">qingjia list</field>
    <field name="model">qingjia.qingjd</field>
    <field name="arch" type="xml">
        <tree>
            <field name="name"/>
            <field name="days"/>
            <field name="startdate"/>
        </tree>
    </field>
</record>
```

##### 修改详情页
```
<record id="action_qingjia_qingjd" model="ir.actions.act_window">
    <field name="name">请假单</field>
    <field name="res_model">qingjia.qingjd</field>
    <field name="view_mode">tree,form</field>
</record>
```
注意：`tree,form`之间不能有空格，不然会报错～～～

也可以对详情页进行定制

```
<record id="action_qingjia_qingjd" model="ir.actions.act_window">
    <field name="name">请假单</field>
    <field name="res_model">qingjia.qingjd</field>
    <field name="arch" type="xml">
        <form>
        <sheet>
        <label for="name"/><field name="name"/>
        <label for="days"/><field name="days"/>
        <label for="startdate"/><field name="startdate"/>
        <label for="reason"/><field name="reason"/>
        </sheet>
        </form>
    </field>
</record>
```
也可以

```
<record id="action_qingjia_qingjd" model="ir.actions.act_window">
    <field name="name">请假单</field>
    <field name="res_model">qingjia.qingjd</field>
    <field name="arch" type="xml">
        <form>
        <group>
            <field name="name"/>
            <field name="days"/>
            <field name="startdate"/>
            <field name="reason"/>
        </group>
        </form>
    </field>
</record>
```
目前这个例子还比较简单，看不出来`group`的作用。

完成以上内容，就可以试着安装一下新模块了。

### 工作流
为了实现工作流，需要对模型进行修改

##### 修改模型

注意 odoo 10 不再支持旧版的api，即

```
    def draft(self, cr, uid, ids, context=None):
        self.write(cr, uid, ids, {'state':'draft'}, context=context)
        return True
```

新版本里，从某种意义上对api进行了一些简化

```
    @api.model
    def draft(self):
        self.write({'state':'draft'})
        return True
```

以下为完整代码

```
class Qingjd(models.Model):
    _name = 'qingjia.qingjd'
    name = fields.Many2one('res.users', string="申请人", required=True)
    manager = fields.Many2one('res.users', string="主管", required=True)
    beginning = fields.Date(string="开始日期", required=True, default=fields.Datetime.now())
    ending = fields.Date(string="结束日期", required=True)
    reason = fields.Text(string="请假事由", required=True)
    accept_reason = fields.Text(string="同意理由", default="同意")

    current_name = fields.Many2one('res.users', string="当前登录人", compute="_get_current_name")
    is_manager = fields.Boolean(compute="_get_is_manager")

    state = fields.Selection([('draft','草稿'),
                              ('confirmed','待审批'),
                              ('accepted','批准'),
                              ('rejected','拒绝'),],
                              string='状态', default='draft', readonly=True)

    @api.model
    def _get_default_name(self):
        uid = self.env.uid
        res = self.env['resource.resource'].search([('user_id', '=', uid)]) 
        name = res.name
        employee = self.env['hr.employee'].search([('name_related','=',name)])
        return employee

    @api.model
    def _get_default_manager(self):
        uid = self.env.uid
        res = self.env['resource.resource'].search([('user_id', '=', uid)])
        name = res.name
        employee = self.env['hr.employee'].search([('name_related','=',name)])
        return employee.parent_id
    

    defaults = {'name':_get_default_name,'manager':_get_default_manager,}

    def _get_is_manager(self):
        if self.current_name == self.manager:
            self.is_manager = True
        else:
            self.is_manager = False

    def _get_current_name(self):
        uid = self.env.uid
        res = self.env['res.users'].search([('id', '=', uid)])
        self.current_name = res

    @api.model
    def draft(self):
        self.write({'state':'draft'})
        return True

    @api.model
    def confirm(self):
        self.write({'state':'confirmed'})
        return True

    @api.model
    def accept(self):
        self.write({'state':'accepted'})
        return True

    @api.model
    def reject(self):
        self.write({'state':'rejected'})
        return True
```

##### 增加视图

```
<record model="ir.ui.view" id="qingjia.qingjia_qingjd_form">
        <field name="name">qing jia dan form</field>
        <field name="model">qingjia.qingjd</field>
        <field name="arch" type="xml">
            <form>
            <header>
                <button name="btn_confirm" type="workflow" states="draft" string="发送" class="oe_highlight"/>
                <button name="btn_accept" type="workflow" states="confirmed" string="批准" class="oe_highlight"/>
                <button name="btn_reject" type="workflow" states="confirmed" string="拒绝" class="oe_highlight"/>
                <field name="state" widget="statusbar" statusbar_visible="draft,confirmed,accepted,rejected" class="oe_highlight" type="workflow"/>
            </header>
                <sheet>
                    <group name="group_top" string="请假单">
                        <group name="group_left">
                            <field name="name"/>
                            <field name="beginning"/>
                        </group>
                        <group name="group_right">
                            <field name="manager"/>
                            <field name="ending"/>
                        </group>
                    </group>
                    <group name="group_below">
                        <field name="reason"/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>
```

##### 增加工作流
```
<odoo>
    <data noupdate="0">
        <record id="wkf_qingjia" model="workflow">
            <field name="name">wkf.qingjia</field>
            <field name="osv">qingjia.qingjd</field>
            <field name="on_create">True</field>
        </record>
        <record id="act_draft" model="workflow.activity">
            <field name="wkf_id" ref="wkf_qingjia"/>
            <field name="name">draft</field>
            <field name="flow_start" eval="True"/>
            <field name="kind">function</field>
            <field name="action">draft()</field>
        </record>
        <record id="act_confirm" model="workflow.activity">
            <field name="wkf_id" ref="wkf_qingjia"/>
            <field name="name">confirm</field>
            <field name="kind">function</field>
            <field name="action">confirm()</field>
        </record>
        <record id="act_accept" model="workflow.activity">
            <field name="wkf_id" ref="wkf_qingjia"/>
            <field name="name">accept</field>
            <field name="kind">function</field>
            <field name="action">accept()</field>
        </record>
        <record id="act_reject" model="workflow.activity">
            <field name="wkf_id" ref="wkf_qingjia"/>
            <field name="name">reject</field>
            <field name="kind">function</field>
            <field name="action">reject()</field>
        </record>
        <record id="qingjia_draft2confirm" model="workflow.transition">
            <field name="act_from" ref="act_draft"/>
            <field name="act_to" ref="act_confirm"/>
            <field name="signal">btn_confirm</field>
        </record>
        <record id="qingjia_confirm2accept" model="workflow.transition">
            <field name="act_from" ref="act_confirm"/>
            <field name="act_to" ref="act_accept"/>
            <field name="signal">btn_accept</field>
            <field name="condition">is_manager</field>
        </record>
        <record id="qingjia_confirm2reject" model="workflow.transition">
            <field name="act_from" ref="act_confirm"/>
            <field name="act_to" ref="act_reject"/>
            <field name="signal">btn_reject</field>
            <field name="condition">is_manager</field>
        </record>
    </data>
</odoo>
```


完整的项目：https://github.com/asdfsx/qingjia