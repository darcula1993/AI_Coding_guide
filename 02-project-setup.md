# AI Coding 指南 - 第二章：项目初始化与架构设计

## 架构先行的重要性

**核心原则：在让AI写代码之前，必须先设计好项目架构**

AI在没有明确架构指导的情况下容易产生耦合度高、难以维护的代码。良好的架构设计是AI Coding成功的基础。

## 项目初始化流程

### 1. 架构设计阶段

#### 与AI协作设计架构

```text
Human: 我要开发一个电商平台的后端API，主要功能包括用户管理、商品管理、订单处理、支付集成。请帮我设计项目架构。

AI: 我建议采用分层架构设计。让我为你设计一个清晰的项目结构：

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

```text
Human: 现在开始实现用户模型。项目使用SQLAlchemy，需要包含基本的用户信息和认证功能。

AI: 我来帮你实现用户模型。基于你的架构，我建议这样设计：
```

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
```

## 数据库初始化策略

### 1. 数据库迁移管理

```python
# app/config/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.config.settings import config

class DatabaseManager:
    """数据库管理器"""
    
    def __init__(self):
        self.engine = create_engine(
            config.database.url,
            echo=config.debug,
            pool_pre_ping=True,
            pool_recycle=300
        )
        self.SessionLocal = sessionmaker(bind=self.engine)
    
    def get_session(self):
        """获取数据库会话"""
        session = self.SessionLocal()
        try:
            yield session
        finally:
            session.close()
    
    def create_tables(self):
        """创建所有表"""
        from app.models import Base
        Base.metadata.create_all(bind=self.engine)
    
    def drop_tables(self):
        """删除所有表（谨慎使用）"""
        from app.models import Base
        Base.metadata.drop_all(bind=self.engine)

# 数据库实例
db_manager = DatabaseManager()
```

### 2. 环境配置文件

```bash
# .env.example
# 数据库配置
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=your_password

# 应用配置
DEBUG=True
SECRET_KEY=your-secret-key-here
JWT_EXPIRATION=3600

# Redis配置
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0

# 邮件配置
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
```

## 容器化配置

### 1. Docker配置

```dockerfile
# Dockerfile
FROM python:3.9-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# 复制requirements文件
COPY requirements.txt .

# 安装Python依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["python", "app.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=postgres
      - DB_NAME=ecommerce
      - DB_USER=postgres
      - DB_PASSWORD=password
    depends_on:
      - postgres
      - redis
    volumes:
      - .:/app
    command: python app.py

  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## API路由架构

### 1. 路由设计模式

```python
# app/routes/__init__.py
from flask import Flask
from app.routes.auth import auth_bp
from app.routes.products import products_bp
from app.routes.orders import orders_bp

def register_routes(app: Flask):
    """注册所有路由"""
    # API版本前缀
    api_prefix = '/api/v1'
    
    app.register_blueprint(auth_bp, url_prefix=f'{api_prefix}/auth')
    app.register_blueprint(products_bp, url_prefix=f'{api_prefix}/products')
    app.register_blueprint(orders_bp, url_prefix=f'{api_prefix}/orders')
```

```python
# app/routes/auth.py
from flask import Blueprint, request, jsonify
from app.services.auth_service import AuthService
from app.utils.decorators import validate_json

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/register', methods=['POST'])
@validate_json(['email', 'password', 'name'])
def register():
    """用户注册"""
    data = request.get_json()
    
    try:
        user = AuthService.register_user(
            email=data['email'],
            password=data['password'],
            name=data['name']
        )
        return jsonify({
            'success': True,
            'user': user.to_dict()
        }), 201
    except ValueError as e:
        return jsonify({
            'success': False,
            'error': str(e)
        }), 400

@auth_bp.route('/login', methods=['POST'])
@validate_json(['email', 'password'])
def login():
    """用户登录"""
    data = request.get_json()
    
    user = AuthService.authenticate_user(
        email=data['email'],
        password=data['password']
    )
    
    if user:
        token = AuthService.generate_token(user.id)
        return jsonify({
            'success': True,
            'token': token,
            'user': user.to_dict()
        })
    else:
        return jsonify({
            'success': False,
            'error': 'Invalid credentials'
        }), 401
```

## 中间件与错误处理

### 1. 自定义装饰器

```python
# app/utils/decorators.py
from functools import wraps
from flask import request, jsonify
import jwt
from app.config.settings import config

def validate_json(required_fields):
    """JSON数据验证装饰器"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not request.is_json:
                return jsonify({'error': 'Content-Type must be application/json'}), 400
            
            data = request.get_json()
            if not data:
                return jsonify({'error': 'No JSON data provided'}), 400
            
            missing_fields = [field for field in required_fields if field not in data]
            if missing_fields:
                return jsonify({
                    'error': f'Missing required fields: {", ".join(missing_fields)}'
                }), 400
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

def require_auth(f):
    """JWT认证装饰器"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        token = request.headers.get('Authorization')
        
        if not token:
            return jsonify({'error': 'Missing Authorization header'}), 401
        
        try:
            if token.startswith('Bearer '):
                token = token[7:]
            
            payload = jwt.decode(token, config.secret_key, algorithms=['HS256'])
            request.current_user_id = payload['user_id']
            
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token has expired'}), 401
        except jwt.InvalidTokenError:
            return jsonify({'error': 'Invalid token'}), 401
        
        return f(*args, **kwargs)
    return decorated_function
```

### 2. 全局错误处理

```python
# app/utils/error_handlers.py
from flask import jsonify
import logging

logger = logging.getLogger(__name__)

class APIError(Exception):
    """自定义API异常"""
    def __init__(self, message, status_code=400, payload=None):
        super().__init__()
        self.message = message
        self.status_code = status_code
        self.payload = payload

def register_error_handlers(app):
    """注册全局错误处理器"""
    
    @app.errorhandler(APIError)
    def handle_api_error(error):
        response = {'error': error.message}
        if error.payload:
            response.update(error.payload)
        return jsonify(response), error.status_code
    
    @app.errorhandler(404)
    def handle_not_found(error):
        return jsonify({'error': 'Resource not found'}), 404
    
    @app.errorhandler(500)
    def handle_internal_error(error):
        logger.error(f'Internal server error: {error}')
        return jsonify({'error': 'Internal server error'}), 500
    
    @app.errorhandler(ValidationError)
    def handle_validation_error(error):
        return jsonify({'error': 'Validation failed', 'details': error.messages}), 400
```

## 日志配置

### 1. 结构化日志

```python
# app/utils/logging_config.py
import logging
import logging.config
from app.config.settings import config

LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
        },
        'detailed': {
            'format': '%(asctime)s [%(levelname)s] %(name)s:%(lineno)d: %(message)s'
        },
        'json': {
            'format': '%(asctime)s %(name)s %(levelname)s %(message)s',
            'class': 'pythonjsonlogger.jsonlogger.JsonFormatter'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
            'stream': 'ext://sys.stdout'
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'INFO',
            'formatter': 'detailed',
            'filename': 'logs/app.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5
        },
        'error_file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'level': 'ERROR',
            'formatter': 'detailed',
            'filename': 'logs/error.log',
            'maxBytes': 10485760,
            'backupCount': 5
        }
    },
    'loggers': {
        '': {  # root logger
            'handlers': ['console', 'file', 'error_file'],
            'level': 'DEBUG' if config.debug else 'INFO',
            'propagate': False
        }
    }
}

def setup_logging():
    """初始化日志配置"""
    import os
    os.makedirs('logs', exist_ok=True)
    logging.config.dictConfig(LOGGING_CONFIG)
```

## 项目启动文件

### 1. 应用程序入口

```python
# app.py
from flask import Flask
from app.config.settings import config
from app.config.database import db_manager
from app.routes import register_routes
from app.utils.error_handlers import register_error_handlers
from app.utils.logging_config import setup_logging

def create_app():
    """应用程序工厂函数"""
    # 初始化日志
    setup_logging()
    
    # 创建Flask应用
    app = Flask(__name__)
    app.config['SECRET_KEY'] = config.secret_key
    app.config['DEBUG'] = config.debug
    
    # 注册路由
    register_routes(app)
    
    # 注册错误处理器
    register_error_handlers(app)
    
    # 初始化数据库
    db_manager.create_tables()
    
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(host='0.0.0.0', port=8000, debug=config.debug)
```

## AI协作开发检查清单

### 1. 项目初始化检查点

- [ ] **架构设计完成** - 明确的分层结构
- [ ] **目录结构创建** - 自动化脚本生成
- [ ] **配置管理就绪** - 环境变量和配置类
- [ ] **数据库连接测试** - 连接池和会话管理
- [ ] **基础模型定义** - 核心数据模型
- [ ] **路由框架搭建** - API路由和蓝图
- [ ] **中间件配置** - 认证、验证、错误处理
- [ ] **日志系统启用** - 结构化日志记录
- [ ] **容器化配置** - Docker和docker-compose
- [ ] **开发环境测试** - 本地启动成功

### 2. 与AI协作的项目设置对话模板

```text
Human: 项目基础架构已经搭建完成，现在需要实现[具体功能模块]。

请按以下步骤协作：
1. 基于现有架构设计数据模型
2. 实现相关的服务层逻辑
3. 创建API控制器
4. 编写单元测试
5. 更新API文档

当前项目使用的技术栈：
- 框架：Flask
- 数据库：PostgreSQL + SQLAlchemy
- 认证：JWT
- 测试：pytest

请从第1步开始，让我们逐步完成这个模块的开发。
```

这种结构化的协作方式确保AI理解项目背景，按既定架构进行开发，避免产生不一致的代码风格。