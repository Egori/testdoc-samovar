# Тестовое задание
## Часть 1: Серверная часть на Go (Backend)  
### 1. Разработка API    

Ссылка на репозиторий с приложением:
https://github.com/Egori/userlist-api-go-test
Использовано ORM Gorm, для миграции задействовано Gorm AutoMigrate.
Написаны тесты для методов сервисного слоя, с мокированием репозитория, а также для тестов с подключением тестовой БД.

### 2. Оптимизация кода

 Оптимизация производительности API для работы с большим объемом данных требует всестороннего подхода, включающего оптимизацию базы данных, кода и инфраструктуры.
####  Оптимизация базы данных
##### Индексация
Необходимо создать индексы в базе данных на колонки, используемые в фильтрации и сортировке, такие как ID, name, Email. Например:
```sql
CREATE INDEX idx_name ON users (name);  
CREATE INDEX idx_email ON users (email);  
CREATE INDEX idx_id ON users (id);  
```
Если используется Gorm,как в нашем случае, индексы прописываются в структуре модели:
```go
type User struct {
	gorm.Model
	Name  string `json:"name" gorm:"size:100;not null;index"`
	Email string `json:"email" gorm:"size:255;not null;unique"`
}
```

При этом индексы будут созданы автоматически в базе данных при создании таблицы.
```go
    db.AutoMigrate(&User{})
```
##### Пагинация
Вместо возврата всех записей в ответе, потребуется реализовать пагинацию. Например:
```go
func (r *userRepository) GetPaginated(limit, offset int) ([]models.User, error) {
    var users []models.User
    err := r.db.Limit(limit).Offset(offset).Find(&users).Error
    return users, err
}
```


