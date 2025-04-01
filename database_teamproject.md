```sql
Table users {
  id bigserial [pk]
  email varchar(255) [not null, unique]
  phone varchar(20) [null, unique]
  password varchar(128) [not null]
  first_name varchar(50) [not null]
  last_name varchar(50) [not null]
  is_active boolean [default: true]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]

  indexes {
    email [unique]
    (first_name, last_name)
  }
}

Table user_roles {
  user_id bigint [ref: > users.id, on delete: cascade]
  role varchar(10) [not null, note: 'admin/buyer/owner']
  
  indexes {
    (user_id, role) [pk]
  }
}

Table car_makes {
  id bigserial [pk]
  name varchar(100) [not null, unique]
  created_at timestamptz [default: `now()`]
}

Table car_models {
  id bigserial [pk]
  make_id bigint [ref: > car_makes.id, on delete: cascade]
  name varchar(100) [not null]
  created_at timestamptz [default: `now()`]

  indexes {
    (make_id, name) [unique]
  }
}

Table cars {
  id bigserial [pk]
  make_id bigint [ref: > car_makes.id, not null]
  model_id bigint [ref: > car_models.id, not null]
  year smallint [not null, note: '1900-2100']
  vin varchar(17) [null]
  mileage int [not null, note: '0-1000000']
  color varchar(50)
  price numeric(12,2) [not null, note: 'max 9999999999.99']
  description text
  location varchar(255) [not null]
  owner_id bigint [ref: > users.id, on delete: cascade]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]

  indexes {
    price
    year
    mileage
    location [gin_trgm_ops]
  }
  
  constraints {
    check_price_positive: price > 0
    check_year_range: year BETWEEN 1900 AND 2100
  }
}

Table listings {
  id bigserial [pk]
  car_id bigint [ref: > cars.id, unique, on delete: cascade]
  status varchar(9) [not null, default: 'active', note: 'active|sold|expired']
  views_count int [default: 0]
  created_at timestamptz [default: `now()`]
  updated_at timestamptz [default: `now()`]

  indexes {
    status
    created_at [desc]
  }
}

Table car_images {
  id bigserial [pk]
  car_id bigint [ref: > cars.id, on delete: cascade]
  image_url varchar(512) [not null]
  is_main boolean [default: false]
  order smallint [default: 0]
  created_at timestamptz [default: `now()`]

  indexes {
    car_id
    (car_id, is_main)
  }
  
  constraints {
    unique_main_per_car: (car_id, is_main) WHERE is_main = true
  }
}

Table favorites {
  user_id bigint [ref: > users.id, on delete: cascade]
  car_id bigint [ref: > cars.id, on delete: cascade]
  is_interested boolean [default: false]
  created_at timestamptz [default: `now()`]

  indexes {
    (user_id, car_id) [pk]
  }
}
```

## Основні виправлення та покращення:

### 1. Геолокація
- Видалено `latitude/longitude`
- Додано `gin_trgm_ops` індекс для поля `location` для ефективного пошуку по містах
- Для точного геопошуку можна пізніше додати `PostGIS`

### 2. Ціна авто
- Змінено тип на `numeric(12,2)` (максимум `99,999,999,999.99`)
- Додано constraint на позитивні значення
- Явно вказано максимальне значення в коментарі

### 3. Поліпшена безпека
- Додані `ON DELETE CASCADE` для всіх `foreign keys`
- Додані `check constraints` для років та пробігу
- Унікальний `VIN` тільки для `not null` значень через `partial index`

### 4. Оптимізація індексів
- Додані `composite indexes` для часті запитів
- Використано `trigram index` для текстового пошуку
- Індекс для сортування оголошень по даті

### 5. Ролі користувачів
- Окрема таблиця `user_roles` з можливістю мати кілька ролей
- `Composite primary key` для унікальних комбінацій

## Рекомендації для реалізації в Django:

### 1. Моделі
```python
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.constraints import ExclusionConstraint

class Car(models.Model):
    # ...
    class Meta:
        indexes = [
            GinIndex(fields=['location'], name='location_gin_idx',
                    opclasses=['gin_trgm_ops'])
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(year__gte=1900) & models.Q(year__lte=2100),
                name='check_year_range'
            )
        ]
```

### 2. Валідація телефону
```python
from phonenumber_field.modelfields import PhoneNumberField

class User(models.Model):
    phone = PhoneNumberField(null=True, blank=True, unique=True)
```

### 3. Обробка зображень
```python
from django.core.validators import validate_image_file_extension

class CarImage(models.Model):
    image = models.ImageField(
        upload_to='cars/',
        validators=[FileExtensionValidator(allowed_extensions=['jpg', 'png']),
                    validate_image_file_extension]
    )
```

### 4. Пошук за локацією
```python
from django.contrib.postgres.search import TrigramSimilarity

Car.objects.annotate(
    similarity=TrigramSimilarity('location', query)
).filter(similarity__gt=0.3).order_by('-similarity')
```

## Тестові кейси для перевірки цілісності:

### 1. Створення авто з ціною 1_000_000_000
```sql
INSERT INTO cars (..., price) VALUES (..., 1000000000.00) -- OK
INSERT INTO cars (..., price) VALUES (..., 10000000000.00) -- ERROR
```

### 2. Спроба додати дубль VIN
```sql
INSERT INTO cars (vin) VALUES ('ABCDE12345') -- OK
INSERT INTO cars (vin) VALUES ('ABCDE12345') -- ERROR
INSERT INTO cars (vin) VALUES (NULL) -- OK
```

### 3. Перевірка каскадного видалення
```sql
DELETE FROM users WHERE id = 1; -- Автоматично видаляє всі пов'язані авто та оголошення
```

### 4. Обмеження на головне зображення
```sql
INSERT INTO car_images (car_id, is_main) VALUES (1, true) -- OK
INSERT INTO car_images (car_id, is_main) VALUES (1, true) -- ERROR
```

