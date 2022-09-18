---
title: "PERM modeling language"
date: 2022-01-25T22:38:50+08:00
draft: false
---

### PML 建模语言

在 PERM 模型中有以下6个原语

1. Request
2. Ploicy
3. Policy Rule
4. Matcher
5. Effect
6. Stub Function

#### Request

request 定义为一对键值对

> request ::= r : attributes
> attributes ::= {attr1, attr2, attr3,...}

request 的键固定为 r,表示需要被鉴权的请求。值是请求实体拥有的属性列表。

一个 request 通常有经典的三元组组成：访问主体(sub)，需要访问的资源(obj)和对资源的动作(act)。
对于这种情况， `attributes = sub, obj, act`。 转换成 PML 的语法就是 `r = sub, obj, act`。

用户使用 PML 可以很灵活的定义 request。 比如 `r = sub, act` 表示不关系具体的资源，`r = tenant, sub, obj, act` 可以适用于多租户的场景

#### Policy

policy 也是一对键值对，定义如下

> policy ::= p : attributes
> attributes ::= {attr1, attr2, attr3,...}

policy的键固定为 p，典型的属性对也是 `sub, obj, act`,PML 的语法为 `p = sub, obj, act`

#### Policy Rule

policy rule 是上面定义的 Policy 的一个实例，是包含一些列值的元组

> policy_rule ::= {value1, value2, value3,...}

元组中元素的个数和 policy 中的定义一致，policy rule 中的值和 policy 中的定义一一绑定。

比如 p = sub, obj, act, policy_rule = (alice, data1, read),绑定关系为 p.sub = alice,
p.obj = data1, p.act = read

#### Matcher

matcher 根据 request 来确定 policy rule 的执行方式。其定义为一个 bool 表达式

> matcher ::=< boolean_expr >(variables, constants, stub_funnction)
> variables ::= {r.attr1, r.attr2,...,p.attr1,p.attr2,...}
> constants ::={const1, connst2, const3,...}

bool 表达式使用操作符将 variablles 和 constants 连起来。操作包括算术（+，-，*，/），关系（==，！=，>, <）和逻辑操作符（&&，||, !）。最简单的 matcher 是 m = r.sub == p.sub && r.obj == p.obj && r.act == p.act。这个 matcher 的意思是 request 和 policy 的 sub,obj，act需要一一对应。

#### Effect

effect 定义如下

> effect ::= <boolean_expr>(effect_term1, effect_term2,...)
> effect_term ::= quantifier, condition
> quantifier ::= some|any|max|min
> condition ::= < expr >(variables, constants, stub_funnction)
> variables ::= {r.attr1, r.attr2,...,p.attr1,p.attr2,...}
> constants ::={const1, connst2, const3,...}

effect 原语决定当多个 policy_rule 满足 request 时，request 是否能够通过。换句话说，
effect 将多个 policy_rule 的匹配结果合并为一个。在单个 effect_term 中，quantifier 将 condition 的有效集合聚合为一个 bool 值。

quantifier 可以是 some，max 或者 min。condition function 和 matcher 类似，但其用于过滤有效的
决定，而不是匹配 request 和 policy_rule。

* some: 只要有一个 policy_rule 匹配，这个 request 就可以通过
* max: 如果有多个 policy_rule 匹配，最终的结果取决于使得 condition 最大的 policy_rule
* min: 如果有多个 policy_rule 匹配，最终的结果取决于使得 condition 最小的 policy_rule

最简单的 allow-override 的 effect 语句如下

> e = some(where (p.eft == allow))

e 代表 effect，eft 是 p 的一个属性，where是关键字。上面的语句意思是只要有一个 policy_rule 匹配，
request 就通过。同理，deny-override 的意思是只要有一个 policy_rule 不匹配，这个 request 就不通过。

> e = !some(where (p.eft == deny))

#### Stub Function

stub function 指的是可以被策略制定者自定义的函数，其定义如下：

> stub-function ::= funnction_name : parameters
> parameters ::= {param1,param2,param3,...}

尽管 PML 表现力已经足够丰富，但是仍然存在很复杂的逻辑，或者检查权限需要查询外部数据。
stub function 允许策略制定者写自定义的逻辑，可以在 matcher 和 effect 中使用。函数
返回类型可以是 boolean, string 或者 integer, stub function 可以和鉴权服务使用同样的语言。

#### 拓展概念

##### Has_Role

在 PML 中，我们不直接定义 role。相反，我们使用 stub function来确定两个属性是否
存在继承关系。 has_role 定义如下：

> has_role ::= function_name: parameters
> function_name ::= "g"
> parameters ::= {attr1,attr2}

has_role 函数典型的使用方式如下

> m = g(r.sub,p.sub) && r.obj == p.obj && r.act == p.act

matcher 检查 request 和 policy_rule 中的角色。在理解 has_role 后，策略制定者能够
存储和检查函数的两个sub的角色是否存在继承关系。通过使用 has_role 函数，PML 能够支持 RBAC1
的场景。

##### Has_Tenant_Role

Has_Tenant_Role 表述了云场景中的多租户，PML 定义了 has_tenant_role 函数，来确定同一租户
的两个属性是否存在角色继承关系。

> has_tenant_role ::= function_name: parameters
> function_name ::= "g"
> parameters ::= {attr1,attr2,tenant}

在 PML 中，角色可以是全局的或者是特定租户下的。
下面的 policy_rule 表示 Alice 在 tenant1 下有 admin 的角色，在 tenant2 下有 user 的角色。
admin 角色可以 (use|manage) 所有资源，user 角色可以 use 所有资源。

> Alice has the admin role in tenant1
> Alice has the user role in tenant2
> admin, *, (use|manage)
> user, *, use

这样 PML 就可以实现相同用户在不同租户下拥有不同的角色。我们可以设计这样的匹配逻辑。
首先判断请求主体是否具有特定租户制定的 policy rule 中的角色。然后匹配 request 中的
obj 和 policy_rule 中的 obj。另外，使用 `|| p.obj == "*"` 满足 policy_rule 中的
通配符。同时自定义 regex_match 函数来匹配 use or manage 的 act。

> m = has_tenant_role(r.sub, p.sub, r.obj.tenant) && (r.obj == p.obj || p.obj == "*") && regex_match(r.act, p.act)

#### 总结

在实际项目中，Policy 和 Matcher 需要事先定义好，Policy 的定义相当于选择权限模型，比如 RBAC，ABAC。以 RBAC 为例，当为角色分配权限时，需要写入一条 policy_rule 的数据。当用户调用后端接口时，
需要重当前的请求中获取 sub, obj， act 等信息，构造出 request。matcher 根据 request 来匹配
policy_rule,聚合出唯一的结果。对于复杂的判断，policy_rule 需要经过 stub_function 预处理，然后
交给 matcher 来处理。


#### 参考

1. [PML: An Interpreter-Based Access Control
Policy Language for Web Services](https://arxiv.org/pdf/1903.09756.pdf)