# AI Coding 指南 - 第五章：模块化开发与代码组织

## 为什么模块化在AI Coding中至关重要

AI在处理大型、耦合度高的代码时容易产生意外的副作用。模块化设计能够：

1. **限制AI的修改范围** - 降低影响其他功能的风险
2. **提高代码可理解性** - AI更容易理解模块的职责
3. **便于单元测试** - 每个模块可独立测试验证
4. **支持并行开发** - 不同模块可同时由AI处理

## 模块化设计原则

### 1. 单一职责原则 (SRP)

每个模块只负责一个明确的功能领域。

#### 示例：用户认证模块

```python
# auth/models.py - 只处理认证相关的数据模型
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional

@dataclass
class AuthToken:
    """认证令牌"""
    token: str
    user_id: int
    expires_at: datetime
    is_active: bool = True
    
    def is_expired(self) -> bool:
        """检查令牌是否过期"""
        return datetime.utcnow() > self.expires_at
    
    def refresh(self, duration_hours: int = 24) -> 'AuthToken':
        """刷新令牌"""
        new_expires = datetime.utcnow() + timedelta(hours=duration_hours)
        return AuthToken(
            token=self.token,
            user_id=self.user_id, 
            expires_at=new_expires,
            is_active=True
        )

# auth/services.py - 只处理认证业务逻辑
import hashlib
import secrets
from typing import Optional, Tuple
from .models import AuthToken
from .exceptions import AuthenticationError, ValidationError

class AuthService:
    """认证服务 - 专注于认证逻辑"""
    
    def __init__(self, token_duration_hours: int = 24):
        self.token_duration_hours = token_duration_hours
    
    def authenticate_user(self, email: str, password: str) -> Optional[AuthToken]:
        """用户认证"""
        # 验证输入
        if not self._validate_email(email):
            raise ValidationError("Invalid email format")
        
        if not self._validate_password(password):
            raise ValidationError("Invalid password format")
        
        # 查找用户（通过依赖注入获取用户服务）
        user = self._get_user_by_email(email)
        if not user or not self._verify_password(password, user.password_hash):
            raise AuthenticationError("Invalid credentials")
        
        # 生成令牌
        return self._generate_token(user.id)
    
    def _generate_token(self, user_id: int) -> AuthToken:
        """生成认证令牌"""
        token = secrets.token_urlsafe(32)
        expires_at = datetime.utcnow() + timedelta(hours=self.token_duration_hours)
        
        return AuthToken(
            token=token,
            user_id=user_id,
            expires_at=expires_at
        )
    
    def _validate_email(self, email: str) -> bool:
        """验证邮箱格式"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    def _validate_password(self, password: str) -> bool:
        """验证密码格式"""
        return len(password) >= 8
    
    def _verify_password(self, password: str, password_hash: str) -> bool:
        """验证密码"""
        # 实际项目中使用bcrypt等专业库
        return hashlib.sha256(password.encode()).hexdigest() == password_hash

# auth/controllers.py - 只处理HTTP请求响应
from flask import Blueprint, request, jsonify
from .services import AuthService
from .exceptions import AuthenticationError, ValidationError

auth_bp = Blueprint('auth', __name__)
auth_service = AuthService()

@auth_bp.route('/login', methods=['POST'])
def login():
    """用户登录接口"""
    try:
        data = request.get_json()
        email = data.get('email')
        password = data.get('password')
        
        token = auth_service.authenticate_user(email, password)
        
        return jsonify({
            'success': True,
            'token': token.token,
            'expires_at': token.expires_at.isoformat()
        }), 200
        
    except ValidationError as e:
        return jsonify({'error': str(e)}), 400
    except AuthenticationError as e:
        return jsonify({'error': str(e)}), 401
    except Exception as e:
        return jsonify({'error': 'Internal server error'}), 500
```

### 2. 依赖注入模式

避免模块间的硬编码依赖，提高可测试性和灵活性。

```python
# services/interfaces.py - 定义服务接口
from abc import ABC, abstractmethod
from typing import Optional, List

class UserRepositoryInterface(ABC):
    """用户仓储接口"""
    
    @abstractmethod
    def get_by_email(self, email: str) -> Optional['User']:
        pass
    
    @abstractmethod
    def create(self, email: str, password_hash: str, name: str) -> 'User':
        pass
    
    @abstractmethod
    def update(self, user_id: int, **kwargs) -> bool:
        pass

class EmailServiceInterface(ABC):
    """邮件服务接口"""
    
    @abstractmethod
    def send_verification_email(self, to_email: str, verification_code: str) -> bool:
        pass

# services/user_service.py - 用户服务实现
class UserService:
    """用户服务 - 通过依赖注入解耦"""
    
    def __init__(self, 
                 user_repository: UserRepositoryInterface,
                 email_service: EmailServiceInterface,
                 auth_service: AuthService):
        self.user_repository = user_repository
        self.email_service = email_service  
        self.auth_service = auth_service
    
    def register_user(self, email: str, password: str, name: str) -> dict:
        """用户注册"""
        # 检查用户是否存在
        existing_user = self.user_repository.get_by_email(email)
        if existing_user:
            raise ValidationError("User already exists")
        
        # 创建用户
        password_hash = self._hash_password(password)
        user = self.user_repository.create(email, password_hash, name)
        
        # 发送验证邮件
        verification_code = self._generate_verification_code()
        self.email_service.send_verification_email(email, verification_code)
        
        # 生成登录令牌
        token = self.auth_service._generate_token(user.id)
        
        return {
            'user_id': user.id,
            'token': token.token,
            'message': 'Registration successful. Please check your email for verification.'
        }
    
    def _hash_password(self, password: str) -> str:
        """密码哈希"""
        import bcrypt
        return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt()).decode('utf-8')
    
    def _generate_verification_code(self) -> str:
        """生成验证码"""
        return secrets.token_hex(6).upper()

# 依赖注入容器配置
class ServiceContainer:
    """服务容器"""
    
    def __init__(self):
        self._services = {}
    
    def register(self, interface, implementation):
        """注册服务"""
        self._services[interface] = implementation
    
    def get(self, interface):
        """获取服务"""
        return self._services.get(interface)

# 配置依赖注入
def configure_services() -> ServiceContainer:
    container = ServiceContainer()
    
    # 注册具体实现
    container.register(UserRepositoryInterface, SQLAlchemyUserRepository())
    container.register(EmailServiceInterface, SMTPEmailService())
    container.register(AuthService, AuthService())
    
    # 注册用户服务
    user_service = UserService(
        container.get(UserRepositoryInterface),
        container.get(EmailServiceInterface), 
        container.get(AuthService)
    )
    container.register(UserService, user_service)
    
    return container
```

## AI协作中的模块化开发流程

### 3. 分模块开发对话示例

```text
Human: 我需要为电商系统添加优惠券功能。请先设计模块结构，然后我们逐步实现。

AI: 好的，让我为优惠券功能设计一个模块化的结构：

**优惠券模块架构：**

```
coupons/
├── __init__.py
├── models/
│   ├── __init__.py
│   ├── coupon.py          # 优惠券数据模型
│   └── usage.py           # 使用记录模型
├── services/
│   ├── __init__.py
│   ├── coupon_service.py  # 优惠券业务逻辑
│   └── validation_service.py # 优惠券验证逻辑
├── repositories/
│   ├── __init__.py
│   └── coupon_repository.py # 数据访问层
├── controllers/
│   ├── __init__.py
│   └── coupon_controller.py # API控制器
├── exceptions/
│   ├── __init__.py
│   └── coupon_exceptions.py # 自定义异常
└── tests/
    ├── __init__.py
    ├── test_models.py
    ├── test_services.py
    └── test_controllers.py
```

**模块职责分配：**
1. **models** - 数据结构定义，不包含业务逻辑
2. **services** - 核心业务逻辑，优惠券计算规则
3. **repositories** - 数据持久化操作
4. **controllers** - HTTP接口处理
5. **exceptions** - 领域特定异常

我们从哪个模块开始实现？我建议先从models开始，这样可以确立数据结构基础。
```

### 4. 逐步实现对话流程

```text
Human: 好的，我们先实现优惠券的数据模型。需要支持百分比折扣和固定金额折扣两种类型。

AI: 明白了，我来设计优惠券的数据模型。我会创建一个灵活的结构来支持不同类型的折扣：

```python
# coupons/models/coupon.py
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Optional, Dict, Any
from decimal import Decimal

class CouponType(Enum):
    """优惠券类型"""
    PERCENTAGE = "percentage"  # 百分比折扣
    FIXED_AMOUNT = "fixed_amount"  # 固定金额折扣

class CouponStatus(Enum):
    """优惠券状态"""
    ACTIVE = "active"
    INACTIVE = "inactive"
    EXPIRED = "expired"

@dataclass
class Coupon:
    """优惠券模型"""
    id: Optional[int]
    code: str  # 优惠券代码
    name: str  # 优惠券名称
    description: str  # 描述
    coupon_type: CouponType  # 折扣类型
    discount_value: Decimal  # 折扣值（百分比或固定金额）
    minimum_order_amount: Optional[Decimal]  # 最小订单金额
    maximum_discount_amount: Optional[Decimal]  # 最大折扣金额
    usage_limit: Optional[int]  # 使用次数限制
    used_count: int = 0  # 已使用次数
    valid_from: datetime  # 有效期开始
    valid_until: datetime  # 有效期结束
    status: CouponStatus = CouponStatus.ACTIVE
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
    
    def is_valid(self) -> bool:
        """检查优惠券是否有效"""
        now = datetime.utcnow()
        return (
            self.status == CouponStatus.ACTIVE and
            self.valid_from <= now <= self.valid_until and
            (self.usage_limit is None or self.used_count < self.usage_limit)
        )
    
    def calculate_discount(self, order_amount: Decimal) -> Decimal:
        """计算折扣金额"""
        if not self.is_valid():
            return Decimal('0')
        
        # 检查最小订单金额
        if (self.minimum_order_amount and 
            order_amount < self.minimum_order_amount):
            return Decimal('0')
        
        if self.coupon_type == CouponType.PERCENTAGE:
            discount = order_amount * (self.discount_value / 100)
        else:  # FIXED_AMOUNT
            discount = self.discount_value
        
        # 应用最大折扣限制
        if self.maximum_discount_amount:
            discount = min(discount, self.maximum_discount_amount)
        
        # 折扣不能超过订单金额
        return min(discount, order_amount)
```

这个模型设计的优点：
1. ✅ 使用枚举类型提高代码安全性
2. ✅ 包含完整的业务逻辑验证
3. ✅ 支持灵活的折扣计算规则
4. ✅ 提供清晰的类型注解
```

## 模块间通信设计

### 5. 事件驱动架构

使用事件来解耦模块间的依赖关系：

```python
# events/base.py - 事件基础设施
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from typing import Any, Dict, List, Callable
import asyncio

@dataclass
class Event:
    """事件基类"""
    event_type: str
    data: Dict[str, Any]
    timestamp: datetime
    event_id: str

class EventHandler(ABC):
    """事件处理器接口"""
    
    @abstractmethod
    async def handle(self, event: Event) -> None:
        pass

class EventBus:
    """事件总线"""
    
    def __init__(self):
        self._handlers: Dict[str, List[EventHandler]] = {}
    
    def subscribe(self, event_type: str, handler: EventHandler):
        """订阅事件"""
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def publish(self, event: Event):
        """发布事件"""
        handlers = self._handlers.get(event.event_type, [])
        
        # 并行处理所有事件处理器
        tasks = [handler.handle(event) for handler in handlers]
        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)

# orders/events.py - 订单相关事件
@dataclass
class OrderCreatedEvent(Event):
    """订单创建事件"""
    
    def __init__(self, order_id: int, user_id: int, total_amount: float):
        super().__init__(
            event_type="order.created",
            data={
                "order_id": order_id,
                "user_id": user_id,
                "total_amount": total_amount
            },
            timestamp=datetime.utcnow(),
            event_id=f"order_created_{order_id}_{int(datetime.utcnow().timestamp())}"
        )

# inventory/handlers.py - 库存模块的事件处理器
class InventoryUpdateHandler(EventHandler):
    """库存更新处理器"""
    
    def __init__(self, inventory_service):
        self.inventory_service = inventory_service
    
    async def handle(self, event: Event) -> None:
        """处理订单创建事件，更新库存"""
        if event.event_type == "order.created":
            order_id = event.data["order_id"]
            # 获取订单详情并更新库存
            await self.inventory_service.update_inventory_for_order(order_id)

# notifications/handlers.py - 通知模块的事件处理器  
class OrderNotificationHandler(EventHandler):
    """订单通知处理器"""
    
    def __init__(self, notification_service):
        self.notification_service = notification_service
    
    async def handle(self, event: Event) -> None:
        """处理订单创建事件，发送通知"""
        if event.event_type == "order.created":
            user_id = event.data["user_id"]
            order_id = event.data["order_id"]
            
            await self.notification_service.send_order_confirmation(
                user_id, order_id
            )

# 事件总线配置
def setup_event_bus() -> EventBus:
    """配置事件总线"""
    event_bus = EventBus()
    
    # 订阅事件处理器
    inventory_handler = InventoryUpdateHandler(inventory_service)
    notification_handler = OrderNotificationHandler(notification_service)
    
    event_bus.subscribe("order.created", inventory_handler)
    event_bus.subscribe("order.created", notification_handler)
    
    return event_bus
```

### 6. 接口隔离原则

为不同的客户端定义专门的接口：

```python
# interfaces/user_interfaces.py
from abc import ABC, abstractmethod
from typing import List, Optional

class UserReadInterface(ABC):
    """用户读取接口 - 只包含查询方法"""
    
    @abstractmethod
    def get_user_profile(self, user_id: int) -> Optional[dict]:
        pass
    
    @abstractmethod
    def search_users(self, query: str) -> List[dict]:
        pass

class UserWriteInterface(ABC):
    """用户写入接口 - 只包含修改方法"""
    
    @abstractmethod
    def update_profile(self, user_id: int, **kwargs) -> bool:
        pass
    
    @abstractmethod
    def deactivate_user(self, user_id: int) -> bool:
        pass

class AdminUserInterface(ABC):
    """管理员用户接口 - 包含管理方法"""
    
    @abstractmethod
    def get_user_statistics(self) -> dict:
        pass
    
    @abstractmethod
    def bulk_update_users(self, user_ids: List[int], **kwargs) -> int:
        pass

# 不同客户端使用不同接口
class UserProfileController:
    """用户资料控制器 - 只依赖读取接口"""
    
    def __init__(self, user_reader: UserReadInterface):
        self.user_reader = user_reader
    
    def get_profile(self, user_id: int):
        return self.user_reader.get_user_profile(user_id)

class UserSettingsController:
    """用户设置控制器 - 需要读写接口"""
    
    def __init__(self, 
                 user_reader: UserReadInterface,
                 user_writer: UserWriteInterface):
        self.user_reader = user_reader
        self.user_writer = user_writer
    
    def update_settings(self, user_id: int, settings: dict):
        # 先读取当前设置
        current_profile = self.user_reader.get_user_profile(user_id)
        if not current_profile:
            return False
        
        # 更新设置  
        return self.user_writer.update_profile(user_id, **settings)
```

## 模块化测试策略

### 7. 模块级单元测试

```python
# tests/test_coupon_service.py
import pytest
from decimal import Decimal
from datetime import datetime, timedelta
from unittest.mock import Mock

from coupons.models.coupon import Coupon, CouponType, CouponStatus
from coupons.services.coupon_service import CouponService
from coupons.exceptions.coupon_exceptions import CouponNotFoundError, CouponExpiredError

class TestCouponService:
    """优惠券服务测试"""
    
    def setup_method(self):
        """测试前准备"""
        self.mock_repository = Mock()
        self.coupon_service = CouponService(self.mock_repository)
    
    def test_apply_percentage_coupon_success(self):
        """测试成功应用百分比优惠券"""
        # 准备测试数据
        coupon = Coupon(
            id=1,
            code="SAVE20",
            name="20% Off",
            description="20% discount",
            coupon_type=CouponType.PERCENTAGE,
            discount_value=Decimal('20'),
            minimum_order_amount=Decimal('100'),
            maximum_discount_amount=Decimal('50'),
            usage_limit=100,
            used_count=0,
            valid_from=datetime.utcnow() - timedelta(days=1),
            valid_until=datetime.utcnow() + timedelta(days=30),
            status=CouponStatus.ACTIVE
        )
        
        self.mock_repository.get_by_code.return_value = coupon
        
        # 执行测试
        order_amount = Decimal('200')
        discount = self.coupon_service.apply_coupon("SAVE20", order_amount)
        
        # 验证结果
        expected_discount = Decimal('40')  # 200 * 20% = 40
        assert discount == expected_discount
        
        # 验证仓储调用
        self.mock_repository.get_by_code.assert_called_once_with("SAVE20")
    
    def test_apply_coupon_with_maximum_discount_limit(self):
        """测试应用有最大折扣限制的优惠券"""
        coupon = Coupon(
            id=1,
            code="SAVE20",
            name="20% Off", 
            description="20% discount",
            coupon_type=CouponType.PERCENTAGE,
            discount_value=Decimal('20'),
            minimum_order_amount=None,
            maximum_discount_amount=Decimal('30'),  # 最大折扣30元
            usage_limit=None,
            used_count=0,
            valid_from=datetime.utcnow() - timedelta(days=1),
            valid_until=datetime.utcnow() + timedelta(days=30),
            status=CouponStatus.ACTIVE
        )
        
        self.mock_repository.get_by_code.return_value = coupon
        
        # 订单金额200，按20%应该折扣40，但最大折扣限制为30
        order_amount = Decimal('200')
        discount = self.coupon_service.apply_coupon("SAVE20", order_amount)
        
        assert discount == Decimal('30')  # 应该被限制为30
    
    def test_apply_expired_coupon_raises_exception(self):
        """测试应用过期优惠券抛出异常"""
        expired_coupon = Coupon(
            id=1,
            code="EXPIRED",
            name="Expired Coupon",
            description="This coupon is expired",
            coupon_type=CouponType.PERCENTAGE,
            discount_value=Decimal('10'),
            minimum_order_amount=None,
            maximum_discount_amount=None,
            usage_limit=None,
            used_count=0,
            valid_from=datetime.utcnow() - timedelta(days=30),
            valid_until=datetime.utcnow() - timedelta(days=1),  # 昨天过期
            status=CouponStatus.ACTIVE
        )
        
        self.mock_repository.get_by_code.return_value = expired_coupon
        
        # 应该抛出过期异常
        with pytest.raises(CouponExpiredError):
            self.coupon_service.apply_coupon("EXPIRED", Decimal('100'))
    
    def test_apply_nonexistent_coupon_raises_exception(self):
        """测试应用不存在的优惠券抛出异常"""
        self.mock_repository.get_by_code.return_value = None
        
        with pytest.raises(CouponNotFoundError):
            self.coupon_service.apply_coupon("NOTFOUND", Decimal('100'))

# pytest运行命令示例
# pytest tests/test_coupon_service.py -v --cov=coupons/services --cov-report=html
```

## 模块化开发的AI协作最佳实践

### 8. 分模块任务分配

```text
Human: 优惠券模块的模型已经完成，现在请实现CouponService业务逻辑层。

**约束条件：**
- 只修改 coupons/services/ 目录下的文件
- 不要修改已有的模型定义
- 依赖注入CouponRepository接口
- 包含完整的异常处理

AI: 好的，我来实现CouponService业务逻辑层。我会严格遵循你的约束条件：

```python
# coupons/services/coupon_service.py
from decimal import Decimal
from typing import Optional, List
from datetime import datetime

from ..models.coupon import Coupon, CouponStatus
from ..repositories.coupon_repository import CouponRepositoryInterface
from ..exceptions.coupon_exceptions import (
    CouponNotFoundError,
    CouponExpiredError, 
    CouponUsageLimitExceededError,
    InsufficientOrderAmountError
)

class CouponService:
    """优惠券业务逻辑服务"""
    
    def __init__(self, coupon_repository: CouponRepositoryInterface):
        self.coupon_repository = coupon_repository
    
    def apply_coupon(self, coupon_code: str, order_amount: Decimal) -> Decimal:
        """
        应用优惠券并返回折扣金额
        
        Args:
            coupon_code: 优惠券代码
            order_amount: 订单金额
            
        Returns:
            Decimal: 折扣金额
            
        Raises:
            CouponNotFoundError: 优惠券不存在
            CouponExpiredError: 优惠券已过期
            CouponUsageLimitExceededError: 超出使用限制
            InsufficientOrderAmountError: 订单金额不足
        """
        # 获取优惠券
        coupon = self.coupon_repository.get_by_code(coupon_code)
        if not coupon:
            raise CouponNotFoundError(f"Coupon '{coupon_code}' not found")
        
        # 验证优惠券状态
        self._validate_coupon(coupon, order_amount)
        
        # 计算折扣
        discount_amount = coupon.calculate_discount(order_amount)
        
        # 更新使用次数（如果有限制）
        if coupon.usage_limit is not None:
            self.coupon_repository.increment_usage_count(coupon.id)
        
        return discount_amount
    
    def _validate_coupon(self, coupon: Coupon, order_amount: Decimal) -> None:
        """验证优惠券是否可用"""
        # 检查状态
        if coupon.status != CouponStatus.ACTIVE:
            raise CouponExpiredError("Coupon is not active")
        
        # 检查时间有效性
        now = datetime.utcnow()
        if now < coupon.valid_from or now > coupon.valid_until:
            raise CouponExpiredError("Coupon is expired or not yet valid")
        
        # 检查使用次数限制
        if (coupon.usage_limit is not None and 
            coupon.used_count >= coupon.usage_limit):
            raise CouponUsageLimitExceededError("Coupon usage limit exceeded")
        
        # 检查最小订单金额
        if (coupon.minimum_order_amount and 
            order_amount < coupon.minimum_order_amount):
            raise InsufficientOrderAmountError(
                f"Order amount must be at least {coupon.minimum_order_amount}"
            )
    
    def get_available_coupons_for_user(self, user_id: int, 
                                     order_amount: Decimal) -> List[Coupon]:
        """获取用户可用的优惠券列表"""
        all_coupons = self.coupon_repository.get_active_coupons()
        available_coupons = []
        
        for coupon in all_coupons:
            try:
                self._validate_coupon(coupon, order_amount)
                available_coupons.append(coupon)
            except Exception:
                # 跳过不可用的优惠券
                continue
        
        return available_coupons
```

这个实现的特点：
1. ✅ 遵循单一职责原则，只处理业务逻辑
2. ✅ 通过依赖注入解耦数据访问层
3. ✅ 完整的异常处理和验证
4. ✅ 清晰的方法职责分离
```

## 模块化设计检查清单

### ✅ 设计阶段

- [ ] 每个模块有明确的单一职责
- [ ] 模块间依赖关系清晰
- [ ] 定义了清晰的接口边界
- [ ] 考虑了模块的可测试性
- [ ] 设计了合适的错误处理策略

### ✅ 实现阶段

- [ ] 遵循依赖注入原则
- [ ] 使用抽象接口而非具体实现
- [ ] 每个模块都有独立的测试
- [ ] 异常处理完整且一致
- [ ] 代码结构清晰易懂

### ✅ AI协作阶段

- [ ] 明确约束AI的修改范围
- [ ] 提供清晰的模块职责说明
- [ ] 逐模块进行开发和测试
- [ ] 定期审查模块间的耦合度
- [ ] 保持接口的稳定性

### ✅ 维护阶段

- [ ] 定期重构高耦合的模块
- [ ] 监控模块的性能表现
- [ ] 更新模块的文档和测试
- [ ] 评估新需求对现有模块的影响

通过遵循这些模块化设计原则，你可以构建出易于维护、测试和扩展的代码库，同时最大化AI协作的效率和安全性。