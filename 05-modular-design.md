# AI Coding 指南 - 第五章：模块化设计的协作策略

## 核心理念：化整为零的智慧

**关键认知：AI在处理小而专一的模块时表现最佳，模块化是AI Coding成功的关键**

大型复杂系统会让AI迷失方向，但清晰的模块边界能帮助AI专注于特定领域，产出更高质量的代码。

## 模块化在AI协作中的独特价值

### 1. 降低AI的认知负担

#### 问题：复杂系统让AI混乱

```text
Human: 帮我优化这个电商系统的性能
```

```text
AI: 我需要了解更多信息...整个系统很复杂，可能涉及：
- 数据库查询优化
- 缓存策略
- 前端性能
- 网络传输
- 服务器配置
- ...
我应该从哪里开始？
```

#### 解决：模块化让AI聚焦

```text
Human: 我需要优化用户认证模块的性能。

模块边界：
- 只涉及auth/目录下的代码
- 主要功能：用户登录、token验证、权限检查
- 不涉及其他业务模块
- 当前问题：登录接口响应慢

请分析auth模块的性能瓶颈。
```

```text
AI: 明确了！让我专注分析用户认证模块的性能问题：

**可能的性能瓶颈：**
1. 密码哈希验证耗时过长
2. 数据库用户查询没有索引
3. JWT token生成算法选择不当
4. 重复的权限检查查询

**建议的优化方向：**
1. 在email字段上添加索引
2. 使用更快的哈希算法（如argon2）
3. 实现JWT token缓存
4. 优化权限检查的查询逻辑

我们先从数据库查询开始优化，你觉得如何？
```

### 2. 提高AI代码质量

#### 分模块的代码审查

```text
Human: 请审查这个支付模块的代码。

模块职责：
- 处理订单支付流程
- 集成第三方支付接口
- 管理支付状态和回调
- 不涉及订单创建和库存管理

```python
# payments/service.py
class PaymentService:
    def __init__(self, payment_gateway):
        self.gateway = payment_gateway
    
    def process_payment(self, order_id, amount, payment_method):
        # 验证订单
        order = Order.get_by_id(order_id)
        if not order:
            raise ValueError("订单不存在")
        
        # 调用支付网关
        result = self.gateway.charge(amount, payment_method)
        
        if result.success:
            # 更新订单状态
            order.status = 'paid'
            order.save()
            
        return result
```

请重点检查模块职责是否单一，以及代码质量。
```

```text
AI: 基于你定义的模块职责，我发现几个问题：

**❌ 职责混乱问题：**
1. 支付服务直接修改订单状态，违反了单一职责原则
2. Order模型的导入和操作应该通过接口隔离

**✅ 改进建议：**

```python
# payments/service.py
class PaymentService:
    def __init__(self, payment_gateway, order_service):
        self.gateway = payment_gateway
        self.order_service = order_service  # 通过依赖注入获取
    
    def process_payment(self, order_id, amount, payment_method):
        # 1. 验证支付请求
        if not self._validate_payment_request(order_id, amount):
            raise ValueError("支付请求无效")
        
        # 2. 调用支付网关
        payment_result = self.gateway.charge(amount, payment_method)
        
        # 3. 记录支付结果
        payment_record = self._create_payment_record(
            order_id, amount, payment_method, payment_result
        )
        
        # 4. 通知订单服务（而不是直接修改）
        if payment_result.success:
            self.order_service.mark_as_paid(order_id, payment_record.id)
        
        return payment_result
    
    def _validate_payment_request(self, order_id, amount):
        # 支付相关的验证逻辑
        return True
    
    def _create_payment_record(self, order_id, amount, method, result):
        # 创建支付记录
        pass
```

这样修改后，支付模块只负责支付相关的逻辑，通过接口与订单模块交互。
```

## AI协作的模块设计模式

### 3. 渐进式模块构建

#### 分阶段的模块开发对话

```text
Human: 我要开发一个优惠券模块。我们分阶段来：

第一阶段：定义模块边界和职责
- 优惠券的创建和管理
- 优惠券的验证逻辑
- 使用记录的追踪
- 不涉及订单计算和支付流程

请先帮我设计模块的基础结构。
```

```text
AI: 好的，让我为优惠券模块设计一个清晰的结构：

**模块目录结构：**
```
coupons/
├── __init__.py
├── models.py          # 数据模型
├── services.py        # 业务逻辑
├── validators.py      # 验证规则
├── exceptions.py      # 自定义异常
└── interfaces.py      # 对外接口定义
```

**模块职责定义：**
1. **models.py** - 优惠券数据结构和基础操作
2. **services.py** - 优惠券业务逻辑（创建、验证、使用）
3. **validators.py** - 优惠券规则验证
4. **exceptions.py** - 优惠券相关的异常处理
5. **interfaces.py** - 其他模块调用的接口

**模块边界：**
- ✅ 管理优惠券的生命周期
- ✅ 验证优惠券的有效性
- ✅ 记录优惠券的使用情况
- ❌ 不处理订单金额计算
- ❌ 不涉及支付流程
- ❌ 不管理用户信息

这个结构符合你的需求吗？我们继续第二阶段的详细设计？
```

```text
Human: 很好！现在进入第二阶段：设计核心数据模型。

请基于以下需求设计优惠券模型：
- 支持满减券（满100减20）
- 支持折扣券（8折）
- 有使用次数限制
- 有有效期限制
- 支持用户限制（每用户限用几次）
```

```text
AI: 基于你的需求，我设计以下数据模型：

```python
# coupons/models.py
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Optional, List
from decimal import Decimal

class CouponType(Enum):
    """优惠券类型"""
    AMOUNT_OFF = "amount_off"  # 满减券
    PERCENT_OFF = "percent_off"  # 折扣券

@dataclass
class Coupon:
    """优惠券模型"""
    id: int
    code: str                    # 优惠券码
    name: str                   # 优惠券名称
    type: CouponType            # 优惠券类型
    
    # 优惠规则
    min_amount: Optional[Decimal] = None      # 最低消费金额
    discount_amount: Optional[Decimal] = None  # 减免金额（满减券）
    discount_percent: Optional[int] = None     # 折扣百分比（折扣券）
    max_discount: Optional[Decimal] = None     # 最大折扣金额
    
    # 使用限制
    total_limit: int = 0          # 总使用次数限制
    user_limit: int = 1           # 每用户使用次数限制
    used_count: int = 0           # 已使用次数
    
    # 有效期
    start_date: datetime
    end_date: datetime
    
    # 状态
    is_active: bool = True
    created_at: datetime = None
    
    def is_valid(self) -> bool:
        """检查优惠券是否有效"""
        now = datetime.utcnow()
        return (
            self.is_active and
            self.start_date <= now <= self.end_date and
            self.used_count < self.total_limit
        )

@dataclass
class CouponUsage:
    """优惠券使用记录"""
    id: int
    coupon_id: int
    user_id: int
    order_id: Optional[int] = None
    discount_amount: Decimal = Decimal('0')
    used_at: datetime = None
```

**设计说明：**
1. 使用枚举定义优惠券类型，类型安全
2. 支持满减和折扣两种模式
3. 灵活的限制机制（总次数+用户次数）
4. 独立的使用记录追踪
5. 清晰的有效性检查逻辑

这个模型设计符合你的需求吗？我们继续设计业务逻辑层？
```

### 4. 接口驱动的模块协作

#### 定义模块间的协作接口

```text
Human: 现在设计优惠券模块与其他模块的协作接口。

需要考虑的协作场景：
1. 订单模块需要验证和应用优惠券
2. 用户模块需要查询用户可用的优惠券
3. 管理模块需要创建和管理优惠券

请设计清晰的接口，避免模块间的紧耦合。
```

```text
AI: 我来设计模块间的协作接口，确保松耦合：

```python
# coupons/interfaces.py
from abc import ABC, abstractmethod
from typing import List, Optional
from decimal import Decimal
from .models import Coupon, CouponUsage

class CouponServiceInterface(ABC):
    """优惠券服务接口 - 供其他模块调用"""
    
    @abstractmethod
    def validate_coupon(self, code: str, user_id: int, order_amount: Decimal) -> dict:
        """验证优惠券
        
        Returns:
            {
                'valid': bool,
                'discount_amount': Decimal,
                'error_message': str
            }
        """
        pass
    
    @abstractmethod
    def apply_coupon(self, code: str, user_id: int, order_id: int, 
                    order_amount: Decimal) -> CouponUsage:
        """应用优惠券"""
        pass
    
    @abstractmethod
    def get_user_available_coupons(self, user_id: int) -> List[Coupon]:
        """获取用户可用的优惠券"""
        pass

class OrderServiceInterface(ABC):
    """订单服务接口 - 优惠券模块需要的外部依赖"""
    
    @abstractmethod
    def get_order_info(self, order_id: int) -> dict:
        """获取订单信息"""
        pass

class UserServiceInterface(ABC):
    """用户服务接口 - 优惠券模块需要的外部依赖"""
    
    @abstractmethod
    def is_valid_user(self, user_id: int) -> bool:
        """验证用户是否有效"""
        pass

# coupons/services.py
class CouponService(CouponServiceInterface):
    """优惠券服务实现"""
    
    def __init__(self, order_service: OrderServiceInterface,
                 user_service: UserServiceInterface):
        self.order_service = order_service
        self.user_service = user_service
    
    def validate_coupon(self, code: str, user_id: int, 
                       order_amount: Decimal) -> dict:
        """验证优惠券"""
        try:
            # 1. 查找优惠券
            coupon = self._get_coupon_by_code(code)
            if not coupon:
                return {'valid': False, 'error_message': '优惠券不存在'}
            
            # 2. 基础有效性检查
            if not coupon.is_valid():
                return {'valid': False, 'error_message': '优惠券已失效'}
            
            # 3. 用户使用次数检查
            user_usage_count = self._get_user_usage_count(coupon.id, user_id)
            if user_usage_count >= coupon.user_limit:
                return {'valid': False, 'error_message': '优惠券使用次数已达上限'}
            
            # 4. 订单金额检查
            if coupon.min_amount and order_amount < coupon.min_amount:
                return {
                    'valid': False, 
                    'error_message': f'订单金额需满{coupon.min_amount}元'
                }
            
            # 5. 计算折扣金额
            discount = self._calculate_discount(coupon, order_amount)
            
            return {
                'valid': True,
                'discount_amount': discount,
                'error_message': ''
            }
            
        except Exception as e:
            return {'valid': False, 'error_message': '系统错误'}
```

**接口设计原则：**
1. **依赖倒置** - 优惠券模块依赖抽象接口，不依赖具体实现
2. **单一职责** - 每个接口只负责一类操作
3. **错误处理** - 统一的错误返回格式
4. **类型安全** - 明确的参数和返回值类型

这样的设计让模块间协作清晰，便于AI理解和修改。
```

## 模块测试的AI协作

### 5. 模块化测试策略

#### 独立测试每个模块

```text
Human: 现在为优惠券模块编写测试用例。

测试范围：
- 只测试优惠券模块的内部逻辑
- 使用mock对象模拟外部依赖
- 覆盖正常流程和异常情况
- 重点测试业务规则的正确性

请先设计测试结构。
```

```text
AI: 我来设计优惠券模块的测试结构：

```python
# tests/test_coupons/
├── __init__.py
├── test_models.py           # 数据模型测试
├── test_services.py         # 业务逻辑测试
├── test_validators.py       # 验证规则测试
├── fixtures.py              # 测试数据
└── mocks.py                # Mock对象

# tests/test_coupons/test_services.py
import pytest
from unittest.mock import Mock, MagicMock
from decimal import Decimal
from datetime import datetime, timedelta
from coupons.services import CouponService
from coupons.models import Coupon, CouponType
from .mocks import MockOrderService, MockUserService

class TestCouponService:
    
    @pytest.fixture
    def service(self):
        """创建测试用的优惠券服务"""
        order_service = MockOrderService()
        user_service = MockUserService()
        return CouponService(order_service, user_service)
    
    @pytest.fixture
    def valid_coupon(self):
        """创建有效的优惠券"""
        return Coupon(
            id=1,
            code="TEST100",
            name="测试优惠券",
            type=CouponType.AMOUNT_OFF,
            min_amount=Decimal('100'),
            discount_amount=Decimal('20'),
            total_limit=100,
            user_limit=1,
            used_count=0,
            start_date=datetime.utcnow() - timedelta(days=1),
            end_date=datetime.utcnow() + timedelta(days=30),
            is_active=True
        )
    
    def test_validate_coupon_success(self, service, valid_coupon):
        """测试优惠券验证成功"""
        # Arrange
        with patch.object(service, '_get_coupon_by_code', return_value=valid_coupon):
            with patch.object(service, '_get_user_usage_count', return_value=0):
                
                # Act
                result = service.validate_coupon("TEST100", 1, Decimal('150'))
                
                # Assert
                assert result['valid'] is True
                assert result['discount_amount'] == Decimal('20')
                assert result['error_message'] == ''
    
    def test_validate_coupon_insufficient_amount(self, service, valid_coupon):
        """测试订单金额不足"""
        # Arrange
        with patch.object(service, '_get_coupon_by_code', return_value=valid_coupon):
            with patch.object(service, '_get_user_usage_count', return_value=0):
                
                # Act
                result = service.validate_coupon("TEST100", 1, Decimal('50'))
                
                # Assert
                assert result['valid'] is False
                assert '订单金额需满100元' in result['error_message']
    
    def test_validate_coupon_usage_limit_exceeded(self, service, valid_coupon):
        """测试使用次数超限"""
        # Arrange
        with patch.object(service, '_get_coupon_by_code', return_value=valid_coupon):
            with patch.object(service, '_get_user_usage_count', return_value=1):
                
                # Act
                result = service.validate_coupon("TEST100", 1, Decimal('150'))
                
                # Assert
                assert result['valid'] is False
                assert '使用次数已达上限' in result['error_message']
```

**测试设计特点：**
1. **隔离性** - 每个测试独立运行，不依赖外部状态
2. **Mock外部依赖** - 使用Mock对象隔离外部模块
3. **全面覆盖** - 正常和异常情况都有测试
4. **清晰结构** - Arrange-Act-Assert模式

这样的测试结构让AI能够专注于模块内部逻辑的验证。
```

## 模块重构的AI协作

### 6. 渐进式模块优化

#### 单模块重构对话

```text
Human: 我发现优惠券验证的性能有问题，每次都要查询数据库。

我想重构优惠券服务，添加缓存机制：
- 只重构CouponService类
- 不改变对外接口
- 添加Redis缓存层
- 保持现有的业务逻辑不变

请帮我设计缓存策略。
```

```text
AI: 我来设计一个不改变接口的缓存策略：

```python
# coupons/services.py (重构版本)
import json
from typing import Optional
from coupons.cache import CouponCache

class CouponService(CouponServiceInterface):
    """带缓存的优惠券服务"""
    
    def __init__(self, order_service: OrderServiceInterface,
                 user_service: UserServiceInterface,
                 cache: Optional[CouponCache] = None):
        self.order_service = order_service
        self.user_service = user_service
        self.cache = cache or CouponCache()
    
    def validate_coupon(self, code: str, user_id: int, 
                       order_amount: Decimal) -> dict:
        """验证优惠券（带缓存）"""
        # 1. 尝试从缓存获取优惠券
        coupon = self.cache.get_coupon(code)
        if not coupon:
            # 缓存未命中，从数据库加载
            coupon = self._get_coupon_by_code(code)
            if coupon:
                self.cache.set_coupon(code, coupon)
        
        if not coupon:
            return {'valid': False, 'error_message': '优惠券不存在'}
        
        # 2. 检查基础有效性（可缓存的部分）
        if not self._is_coupon_basically_valid(coupon):
            return {'valid': False, 'error_message': '优惠券已失效'}
        
        # 3. 检查用户使用次数（实时查询，不适合缓存）
        user_usage_count = self._get_user_usage_count(coupon.id, user_id)
        if user_usage_count >= coupon.user_limit:
            return {'valid': False, 'error_message': '优惠券使用次数已达上限'}
        
        # 4. 其他验证逻辑保持不变
        if coupon.min_amount and order_amount < coupon.min_amount:
            return {
                'valid': False, 
                'error_message': f'订单金额需满{coupon.min_amount}元'
            }
        
        discount = self._calculate_discount(coupon, order_amount)
        return {
            'valid': True,
            'discount_amount': discount,
            'error_message': ''
        }

# coupons/cache.py (新增)
import redis
import json
from typing import Optional
from .models import Coupon

class CouponCache:
    """优惠券缓存管理"""
    
    def __init__(self, redis_client=None, ttl=3600):
        self.redis = redis_client or redis.Redis()
        self.ttl = ttl  # 缓存过期时间（秒）
    
    def get_coupon(self, code: str) -> Optional[Coupon]:
        """从缓存获取优惠券"""
        try:
            key = f"coupon:{code}"
            data = self.redis.get(key)
            if data:
                coupon_dict = json.loads(data)
                return Coupon(**coupon_dict)
            return None
        except Exception:
            # 缓存异常时直接返回None，降级到数据库查询
            return None
    
    def set_coupon(self, code: str, coupon: Coupon):
        """设置优惠券缓存"""
        try:
            key = f"coupon:{code}"
            # 只缓存相对稳定的数据，不缓存used_count等动态数据
            cache_data = {
                'id': coupon.id,
                'code': coupon.code,
                'name': coupon.name,
                'type': coupon.type.value,
                'min_amount': str(coupon.min_amount) if coupon.min_amount else None,
                'discount_amount': str(coupon.discount_amount) if coupon.discount_amount else None,
                'discount_percent': coupon.discount_percent,
                'max_discount': str(coupon.max_discount) if coupon.max_discount else None,
                'total_limit': coupon.total_limit,
                'user_limit': coupon.user_limit,
                'start_date': coupon.start_date.isoformat(),
                'end_date': coupon.end_date.isoformat(),
                'is_active': coupon.is_active
            }
            self.redis.setex(key, self.ttl, json.dumps(cache_data))
        except Exception:
            # 缓存失败不影响主流程
            pass
    
    def invalidate_coupon(self, code: str):
        """使优惠券缓存失效"""
        try:
            key = f"coupon:{code}"
            self.redis.delete(key)
        except Exception:
            pass
```

**重构策略说明：**
1. **接口兼容** - 对外接口完全不变
2. **优雅降级** - 缓存失败时自动降级到数据库查询
3. **选择性缓存** - 只缓存相对稳定的数据
4. **失效机制** - 提供缓存失效的接口

这个重构保持了模块的边界清晰，AI很容易理解和维护。
```

## 模块化协作检查清单

### 设计阶段检查
- [ ] **职责边界明确** - 每个模块只负责一个领域
- [ ] **接口定义清晰** - 模块间通过接口交互
- [ ] **依赖关系简单** - 避免循环依赖和过度耦合
- [ ] **数据模型独立** - 每个模块有自己的数据结构

### 实现阶段检查
- [ ] **目录结构清晰** - 文件组织符合模块职责
- [ ] **命名规范一致** - 使用统一的命名约定
- [ ] **错误处理完善** - 模块内部和接口层都有错误处理
- [ ] **文档说明充分** - 接口和关键逻辑有清晰说明

### 测试阶段检查
- [ ] **单元测试覆盖** - 每个模块有独立的测试
- [ ] **集成测试验证** - 模块间协作正确
- [ ] **Mock对象使用** - 隔离外部依赖进行测试
- [ ] **边界条件测试** - 异常情况处理正确

### AI协作检查
- [ ] **模块边界沟通** - 向AI明确说明模块职责
- [ ] **接口契约确认** - AI理解模块间的协作方式
- [ ] **变更范围限制** - 修改时明确影响范围
- [ ] **渐进式开发** - 分阶段完成模块功能

**记住**：好的模块化设计是AI Coding成功的基石。清晰的模块边界让AI能够专注于特定领域，产出更高质量、更可维护的代码。