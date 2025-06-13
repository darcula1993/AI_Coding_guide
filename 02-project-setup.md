# AI Coding 指南 - 第二章：项目初始化与架构设计

## 架构先行的重要性

**核心原则：在让AI写代码之前，必须先设计好项目架构**

AI在没有明确架构指导的情况下容易产生耦合度高、难以维护的代码。良好的架构设计是AI Coding成功的基础。

## 项目初始化流程

### 1. 架构设计阶段

#### 与AI协作设计架构

```
Human: 我要开发一个电商平台的后端API，主要功能包括用户管理、商品管理、订单处理、支付集成。请帮我设计项目架构。

AI: 我建议采用分层架构设计。让我为你设计一个清晰的项目结构：

```
ecommerce-backend/
├── app/
│   ├── __init__.py
│   ├── models/          # 数据模型层
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── product.py
│   │   └── order.py
│   ├── services/        # 业务逻辑层
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── product_service.py
│   │   └── order_service.py
│   ├── controllers/     # 控制器层
│   │   ├── __init__.py
│   │   ├── auth_controller.py
│   │   ├── product_controller.py
│   │   └── order_controller.py
│   ├── utils/          # 工具函数
│   │   ├── __init__.py
│   │   ├── validators.py
│   │   └── helpers.py
│   └── config/         # 配置管理
│       ├── __init__.py
│       ├── settings.py
│       └── database.py
├── tests/              # 测试目录
├── docs/               # 文档
├── requirements.txt
└── README.md
```

这种设计的优势是：
1. 清晰的职责分离
2. 便于单元测试
3. 易于扩展和维护
4. 符合SOLID原则
```

### 2. 目录结构创建

#### 使用AI生成项目脚手架

```python
# project_setup_script.py
import os

def create_project_structure():
    """
    根据设计的架构创建项目目录结构
    """
    directories = [
        'app',
        'app/models',
        'app/services', 
        'app/controllers',
        'app/utils',
        'app/config',
        'tests',
        'tests/unit',
        'tests/integration',
        'docs'
    ]
    
    for directory in directories:
        os.makedirs(directory, exist_ok=True)
        # 创建__init__.py文件
        if directory.startswith('app'):
            init_file = os.path.join(directory, '__init__.py')
            with open(init_file, 'w') as f:
                f.write(f'"""{"Module: " + directory.replace("app/", "")}"""\n')
    
    print("项目结构创建完成!")

if __name__ == "__main__":
    create_project_structure()
```

### 3. 配置管理

#### 环境配置设计

```python
# app/config/settings.py
import os
from typing import Optional
from dataclasses import dataclass

@dataclass
class DatabaseConfig:
    """数据库配置"""
    host: str
    port: int
    name: str
    user: str
    password: str
    
    @property
    def url(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"

@dataclass 
class Config:
    """应用程序配置"""
    debug: bool = False
    secret_key: str = ""
    database: Optional[DatabaseConfig] = None
    jwt_expiration: int = 3600  # 1 hour
    
    @classmethod
    def from_env(cls) -> 'Config':
        """从环境变量加载配置"""
        db_config = DatabaseConfig(
            host=os.getenv('DB_HOST', 'localhost'),
            port=int(os.getenv('DB_PORT', '5432')),
            name=os.getenv('DB_NAME', 'ecommerce'),
            user=os.getenv('DB_USER', 'postgres'),
            password=os.getenv('DB_PASSWORD', '')
        )
        
        return cls(
            debug=os.getenv('DEBUG', 'False').lower() == 'true',
            secret_key=os.getenv('SECRET_KEY', ''),
            database=db_config,
            jwt_expiration=int(os.getenv('JWT_EXPIRATION', '3600'))
        )

# 使用示例
config = Config.from_env()
```

## 模块化设计原则

### 1. 单一职责原则

每个模块只负责一个功能领域：

```python
# app/services/auth_service.py
from typing import Optional
from app.models.user import User
from app.utils.validators import validate_email
from app.utils.crypto import hash_password, verify_password

class AuthService:
    """用户认证服务 - 只负责认证相关业务逻辑"""
    
    @staticmethod
    def register_user(email: str, password: str, name: str) -> User:
        """用户注册"""
        # 输入验证
        if not validate_email(email):
            raise ValueError("Invalid email format")
        
        if len(password) < 8:
            raise ValueError("Password must be at least 8 characters")
        
        # 检查用户是否已存在
        if User.get_by_email(email):
            raise ValueError("User already exists")
        
        # 创建用户
        hashed_password = hash_password(password)
        user = User.create(
            email=email,
            password=hashed_password,
            name=name
        )
        
        return user
    
    @staticmethod
    def authenticate_user(email: str, password: str) -> Optional[User]:
        """用户认证"""
        user = User.get_by_email(email)
        if not user:
            return None
        
        if verify_password(password, user.password):
            return user
        
        return None
```

### 2. 依赖注入模式

```python
# app/services/order_service.py
from typing import List
from app.models.order import Order
from app.services.payment_service import PaymentService
from app.services.inventory_service import InventoryService

class OrderService:
    """订单服务"""
    
    def __init__(self, 
                 payment_service: PaymentService,
                 inventory_service: InventoryService):
        self.payment_service = payment_service
        self.inventory_service = inventory_service
    
    def create_order(self, user_id: int, items: List[dict]) -> Order:
        """创建订单"""
        # 检查库存
        for item in items:
            if not self.inventory_service.check_availability(
                item['product_id'], item['quantity']
            ):
                raise ValueError(f"Product {item['product_id']} not available")
        
        # 计算总价
        total_amount = self._calculate_total(items)
        
        # 创建订单
        order = Order.create(
            user_id=user_id,
            items=items,
            total_amount=total_amount,
            status='pending'
        )
        
        return order
    
    def _calculate_total(self, items: List[dict]) -> float:
        """计算订单总价"""
        total = 0
        for item in items:
            product = Product.get_by_id(item['product_id'])
            total += product.price * item['quantity']
        return total
```

## AI协作最佳实践

### 1. 分阶段开发对话

```
Human: 现在开始实现用户模型。项目使用SQLAlchemy，需要包含基本的用户信息和认证功能。

AI: 我来帮你实现用户模型。基于你的架构，我建议这样设计：

```python
# app/models/user.py
from datetime import datetime
from typing import Optional
from sqlalchemy import Column, Integer, String, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base
from werkzeug.security import generate_password_hash, check_password_hash

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    
    id = Column(Integer, primary_key=True)
    email = Column(String(120), unique=True, nullable=False, index=True)
    name = Column(String(80), nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    def set_password(self, password: str) -> None:
        """设置密码"""
        self.password_hash = generate_password_hash(password)
    
    def check_password(self, password: str) -> bool:
        """验证密码"""
        return check_password_hash(self.password_hash, password)
    
    @classmethod
    def get_by_email(cls, email: str) -> Optional['User']:
        """根据邮箱获取用户"""
        # 这里需要数据库session，在实际项目中通过依赖注入获取
        pass
    
    def to_dict(self) -> dict:
        """转换为字典格式"""
        return {
            'id': self.id,
            'email': self.email,
            'name': self.name,
            'is_active': self.is_active,
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat()
        }
```

这个模型的特点：
1. 遵循了你的命名约定
2. 包含了基础的认证功能
3. 提供了便捷的查询方法
4. 支持数据序列化