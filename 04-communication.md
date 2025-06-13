# AI Coding 指南 - 第四章：高效沟通的艺术

## 核心理念：AI是你的编程伙伴

**关键认知：AI不是搜索引擎，而是需要上下文和反馈的协作伙伴**

与AI协作编程最大的误区是把它当作"万能的代码生成器"。实际上，AI更像一个有经验但需要明确指导的开发者。

## 沟通模式的转变

### 1. 从命令式到协作式

#### 错误的沟通方式

```text
Human: 写一个登录功能
```

```text
AI: 好的，这是一个登录功能的实现...
[直接输出大量代码]
```

**问题**：缺乏上下文，AI只能提供通用解决方案，不符合具体项目需求。

#### 正确的协作方式

```text
Human: 我需要为现有项目添加用户登录功能。让我先说明一下项目情况：

项目背景：
- 使用Flask + SQLAlchemy
- 已有User模型，包含email和password字段
- 需要JWT认证方式
- 前端是React，需要JSON API

具体需求：
- 用户用邮箱+密码登录
- 登录成功返回JWT token
- 错误时返回明确的错误信息

你能帮我设计这个登录接口吗？实现前我们先讨论一下方案。
```

```text
AI: 好的，基于你的项目情况，我建议这样设计登录接口：

**接口设计：**
- POST /api/auth/login
- 接收: {"email": "xxx", "password": "xxx"}
- 成功返回: {"token": "jwt_token", "user": {"id": 1, "email": "xxx"}}
- 失败返回: {"error": "错误信息"}

**主要步骤：**
1. 验证邮箱格式
2. 查找用户记录
3. 验证密码
4. 生成JWT token
5. 返回响应

**几个确认问题：**
1. JWT的过期时间你希望设置多长？
2. 密码验证失败时，是否需要记录尝试次数？
3. 是否需要"记住我"功能？

这个方案符合你的需求吗？
```

### 2. 渐进式需求澄清

#### 分步骤明确需求

```text
Human: 现在我们确定登录方案了，开始实现。

JWT过期时间设置为24小时，暂时不需要记录失败次数和"记住我"功能。

请先实现登录的核心逻辑，我们一步步来。
```

```text
AI: 好的，我先实现登录的核心逻辑。基于你的需求，我会创建一个login函数：

```python
from flask import request, jsonify
from werkzeug.security import check_password_hash
import jwt
from datetime import datetime, timedelta
from your_app.models import User
from your_app import app

@app.route('/api/auth/login', methods=['POST'])
def login():
    try:
        data = request.get_json()
        
        # 验证输入数据
        email = data.get('email')
        password = data.get('password')
        
        if not email or not password:
            return jsonify({'error': '邮箱和密码不能为空'}), 400
        
        # 查找用户
        user = User.query.filter_by(email=email).first()
        if not user:
            return jsonify({'error': '用户不存在'}), 401
        
        # 验证密码
        if not check_password_hash(user.password, password):
            return jsonify({'error': '密码错误'}), 401
        
        # 生成JWT token
        payload = {
            'user_id': user.id,
            'email': user.email,
            'exp': datetime.utcnow() + timedelta(hours=24)
        }
        token = jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')
        
        # 返回成功响应
        return jsonify({
            'token': token,
            'user': {
                'id': user.id,
                'email': user.email
            }
        }), 200
        
    except Exception as e:
        return jsonify({'error': '服务器内部错误'}), 500
```

这个实现包含了我们讨论的所有要点。你想先测试一下这个版本，还是有什么地方需要调整？
```

## 上下文管理策略

### 3. 项目上下文的建立

#### 标准化的项目信息模板

```text
Human: 我要开始一个新功能的开发，先让我介绍一下项目情况：

**项目基本信息：**
- 项目类型：电商后端API
- 技术栈：Python 3.9 + Flask + PostgreSQL
- 认证方式：JWT
- API风格：RESTful
- 当前阶段：已完成用户认证，正在开发商品管理

**代码结构：**
```
app/
├── models/
│   ├── user.py
│   └── product.py
├── routes/
│   ├── auth.py
│   └── products.py
└── utils/
    └── auth.py
```

**当前任务：**
需要为商品管理添加分类功能，用户可以按分类浏览商品。

基于这个背景，你能帮我设计一下商品分类的数据模型吗？
```

```text
AI: 基于你的项目情况，我来设计商品分类的数据模型。

考虑到电商场景，我建议支持多级分类结构，比如：
- 电子产品 > 手机 > 智能手机
- 服装 > 男装 > T恤

**数据模型设计：**

```python
# app/models/category.py
from app import db
from datetime import datetime

class Category(db.Model):
    __tablename__ = 'categories'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    slug = db.Column(db.String(100), unique=True, nullable=False)
    description = db.Column(db.Text)
    
    # 支持多级分类
    parent_id = db.Column(db.Integer, db.ForeignKey('categories.id'))
    parent = db.relationship('Category', remote_side=[id], backref='children')
    
    # 排序和状态
    sort_order = db.Column(db.Integer, default=0)
    is_active = db.Column(db.Boolean, default=True)
    
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'name': self.name,
            'slug': self.slug,
            'description': self.description,
            'parent_id': self.parent_id,
            'sort_order': self.sort_order,
            'is_active': self.is_active
        }
```

然后需要更新Product模型来建立关联关系。你觉得这个设计怎么样？有什么需要调整的地方吗？
```

### 4. 错误上下文的有效传递

#### 完整的错误报告模板

```text
Human: 我遇到了一个问题，需要你帮忙分析。

**问题描述：**
用户登录时偶尔出现JWT token无效的错误，但重新登录就正常了。

**错误信息：**
```
jwt.exceptions.InvalidTokenError: Signature verification failed
```

**复现步骤：**
1. 用户正常登录获取token
2. 使用token访问需要认证的接口
3. 大约10-15分钟后，token突然变成无效
4. 重新登录后恢复正常

**环境信息：**
- 服务器：Ubuntu 20.04
- Python 3.9
- JWT库：PyJWT 2.4.0
- 部署方式：Gunicorn + Nginx

**相关代码：**
```python
# token生成
payload = {
    'user_id': user.id,
    'exp': datetime.utcnow() + timedelta(hours=24)
}
token = jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')

# token验证
try:
    payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
except jwt.InvalidTokenError:
    return jsonify({'error': 'Invalid token'}), 401
```

你觉得可能是什么原因？
```

```text
AI: 基于你提供的信息，我分析了几个可能的原因：

**最可能的原因：服务器时间同步问题**

问题出现在"10-15分钟后"这个时间点，很可能是服务器时间不同步导致的。JWT的exp字段使用UTC时间，如果服务器时间有偏差，就会导致token提前过期。

**排查方法：**
1. 检查服务器时间同步：
```bash
timedatectl status
# 如果NTP同步关闭，执行：
sudo timedatectl set-ntp true
```

2. 在JWT生成时添加调试信息：
```python
from datetime import datetime
import pytz

# 生成token时记录时间
current_time = datetime.utcnow()
expire_time = current_time + timedelta(hours=24)

print(f"Token生成时间: {current_time}")
print(f"Token过期时间: {expire_time}")

payload = {
    'user_id': user.id,
    'iat': current_time,  # 添加签发时间
    'exp': expire_time
}
```

**其他可能原因：**
- Gunicorn多进程间SECRET_KEY不一致
- 负载均衡器导致的时间差异
- JWT库版本兼容性问题

你可以先检查一下服务器时间同步，然后告诉我结果。我们可以进一步排查。
```

## 需求澄清的沟通技巧

### 5. 功能边界的明确定义

#### 范围限定对话

```text
Human: 我需要给电商系统添加优惠券功能。

但是我们先明确一下范围，这次只做最基础的功能：
- 只支持满减优惠券（满100减20这种）
- 只有管理员可以创建优惠券
- 用户可以在下单时输入优惠券码使用
- 每个优惠券有使用次数限制

暂时不需要：
- 折扣券、赠品券等其他类型
- 用户优惠券领取功能
- 复杂的叠加规则

基于这个范围，你觉得需要哪些数据表？
```

```text
AI: 基于你定义的功能范围，我建议创建以下数据表：

**1. 优惠券表 (coupons)**
```python
class Coupon(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    code = db.Column(db.String(50), unique=True, nullable=False)  # 优惠券码
    name = db.Column(db.String(100), nullable=False)  # 优惠券名称
    
    # 满减条件
    min_amount = db.Column(db.Decimal(10, 2), nullable=False)  # 满多少
    discount_amount = db.Column(db.Decimal(10, 2), nullable=False)  # 减多少
    
    # 使用限制
    total_count = db.Column(db.Integer, nullable=False)  # 总发放量
    used_count = db.Column(db.Integer, default=0)  # 已使用量
    
    # 有效期
    start_date = db.Column(db.DateTime, nullable=False)
    end_date = db.Column(db.DateTime, nullable=False)
    
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
```

**2. 优惠券使用记录表 (coupon_usages)**
```python
class CouponUsage(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    coupon_id = db.Column(db.Integer, db.ForeignKey('coupons.id'))
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))
    order_id = db.Column(db.Integer, db.ForeignKey('orders.id'))
    discount_amount = db.Column(db.Decimal(10, 2))  # 实际优惠金额
    used_at = db.Column(db.DateTime, default=datetime.utcnow)
```

这样的设计能满足你的需求吗？还需要考虑其他字段吗？
```

### 6. 技术实现的选择讨论

#### 方案对比对话

```text
Human: 对于优惠券验证逻辑，我在考虑两种实现方式：

方案A：在下单时验证和扣减
方案B：先锁定优惠券，支付成功后确认使用

你觉得哪种方式更好？我们的业务场景是小型电商，并发不高。
```

```text
AI: 对于你的业务场景，我推荐**方案A：下单时验证和扣减**。

**推荐理由：**

1. **简单直接**：小型电商并发不高，不需要复杂的锁定机制
2. **减少状态管理**：避免处理锁定超时、释放等复杂逻辑
3. **用户体验好**：下单即确定优惠，无需担心支付时优惠券被占用

**实现方案：**
```python
def apply_coupon(order_data, coupon_code):
    # 1. 验证优惠券
    coupon = Coupon.query.filter_by(code=coupon_code, is_active=True).first()
    if not coupon:
        raise ValueError("优惠券不存在或已失效")
    
    # 2. 检查使用次数（原子操作）
    result = db.session.query(Coupon).filter(
        Coupon.id == coupon.id,
        Coupon.used_count < Coupon.total_count
    ).update({
        'used_count': Coupon.used_count + 1
    })
    
    if result == 0:
        raise ValueError("优惠券已被用完")
    
    # 3. 验证订单金额
    if order_data['total_amount'] < coupon.min_amount:
        # 回滚使用次数
        coupon.used_count -= 1
        db.session.commit()
        raise ValueError(f"订单金额需满{coupon.min_amount}元")
    
    # 4. 计算优惠金额
    discount = min(coupon.discount_amount, order_data['total_amount'])
    
    return discount
```

如果后续业务增长需要更复杂的并发控制，我们再升级到方案B。你觉得这个方案怎么样？
```

## 代码审查的协作对话

### 7. 结构化的代码Review

#### 分层审查对话

```text
Human: 我需要你帮我审查这段优惠券验证的代码。我们分几个层面来看：

第一层：基本逻辑和语法
第二层：业务逻辑正确性
第三层：安全性和边界情况
第四层：性能和可维护性

我们一层层来，先看第一层。

```python
def validate_coupon(coupon_code, order_amount):
    coupon = Coupon.query.filter_by(code=coupon_code).first()
    if coupon and coupon.is_active:
        if order_amount >= coupon.min_amount:
            if coupon.used_count < coupon.total_count:
                if coupon.start_date <= datetime.now() <= coupon.end_date:
                    return True, coupon.discount_amount
    return False, 0
```
```

```text
AI: **第一层审查：基本逻辑和语法**

✅ **语法正确**：代码可以正常运行，没有语法错误

❌ **逻辑结构问题**：
1. 嵌套层次过深，可读性差
2. 没有明确的错误信息返回
3. datetime.now()应该使用datetime.utcnow()保持一致性

**建议改进：**
```python
def validate_coupon(coupon_code, order_amount):
    coupon = Coupon.query.filter_by(code=coupon_code).first()
    
    if not coupon:
        return False, 0, "优惠券不存在"
    
    if not coupon.is_active:
        return False, 0, "优惠券已停用"
    
    current_time = datetime.utcnow()
    if not (coupon.start_date <= current_time <= coupon.end_date):
        return False, 0, "优惠券不在有效期内"
    
    if order_amount < coupon.min_amount:
        return False, 0, f"订单金额需满{coupon.min_amount}元"
    
    if coupon.used_count >= coupon.total_count:
        return False, 0, "优惠券已被用完"
    
    return True, coupon.discount_amount, "验证通过"
```

第一层审查完成。你觉得这些建议怎么样？我们继续第二层业务逻辑审查吗？
```

## 沟通效率优化

### 8. 标准化术语和模式

#### 建立项目词汇表

```text
Human: 为了提高我们的沟通效率，我想建立一套项目标准术语。

**数据实体术语：**
- 用户 = User（注册用户，有完整账户信息）
- 访客 = Visitor（未注册或未登录用户）
- 商品 = Product（可销售的商品项）
- 订单 = Order（用户的购买记录）
- 优惠券 = Coupon（满减优惠券）

**操作术语：**
- 下单 = 创建订单，但未支付
- 支付 = 完成付款流程
- 发货 = 商家发出商品
- 确认收货 = 用户确认收到商品

**状态术语：**
- 待支付 = pending_payment
- 已支付 = paid
- 已发货 = shipped
- 已完成 = completed
- 已取消 = cancelled

以后我们的对话都使用这套术语，这样你能更准确理解我的需求，可以吗？
```

```text
AI: 完全同意！建立标准术语很重要，我已经记住了这套定义。

为了确保理解准确，我来确认几个要点：

1. **用户 vs 访客**：访客可以浏览商品但不能下单，对吗？
2. **下单 vs 支付**：下单后订单状态是pending_payment，支付成功后变为paid？
3. **商品状态**：商品本身是否也有状态（上架/下架）？

另外建议补充几个术语：
- **库存** = 商品的可售数量
- **购物车** = 用户临时存放待购商品的容器
- **退款** = 已支付订单的金额返还

这样我们就有了完整的术语体系。以后的对话中，我会严格使用这些术语，确保沟通精确。
```

## AI Coding沟通检查清单

### 开始对话前准备
- [ ] **明确项目背景** - 技术栈、当前状态、业务场景
- [ ] **准备相关代码** - 现有实现、错误日志、配置信息
- [ ] **定义需求边界** - 功能范围、限制条件、优先级
- [ ] **建立术语标准** - 统一关键概念的表达方式

### 对话过程中维护
- [ ] **逐步澄清需求** - 避免一次性复杂需求描述
- [ ] **提供充分上下文** - 让AI理解完整的业务场景
- [ ] **监控AI理解** - 确认AI的分析方向正确
- [ ] **明确修改范围** - 限制变更的影响范围

### 代码生成后验证
- [ ] **逻辑正确性审查** - 业务逻辑是否符合需求
- [ ] **安全性检查** - 是否存在安全漏洞
- [ ] **边界条件测试** - 异常情况处理是否完善
- [ ] **集成兼容性** - 是否与现有代码兼容

**记住**：与AI的高效沟通是一门艺术，需要在实践中不断优化。好的沟通能让AI成为你最得力的编程伙伴。