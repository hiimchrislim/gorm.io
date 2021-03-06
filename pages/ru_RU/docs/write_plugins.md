---
title: Написание плагинов
layout: страница
---

## Callbacks

Сам GORM основан на методах `Callbacks`, у которого есть callback функции для `Create`, `Query`, `Update`, `Delete`, `Row`, `Raw`, с помощью которых вы можете полностью настроить GORM по своему усмотрению

Вызовы callback регистрируются в глобальной `*gorm.DB`, а не на сессионном уровне, если вам требуется `*gorm. B` с другими функциями callback, вам нужно инициализировать другой `*gorm.DB`

### Регистрация функций callback

Регистрация функции callback в Callbacks

```go
func cropImage(db *gorm.DB) {
  if db.Statement.Schema != nil {
    // обрезка изображений и загрузка их на сервер CDN, пример кода
    for _, field := range db.Statement.Schema.Fields {
      switch db.Statement.ReflectValue.Kind() {
      case reflect.Slice, reflect.Array:
        for i := 0; i < db.Statement.ReflectValue.Len(); i++ {
          if fieldValue, isZero := field.ValueOf(db.Statement.ReflectValue.Index(i)); !isZero {
            if crop, ok := fieldValue.(CropInterface); ok {
              crop.Crop()
            }
          }
        }
      case reflect.Struct:
        if fieldValue, isZero := field.ValueOf(db.Statement.ReflectValue.Index(i)); isZero {
          if crop, ok := fieldValue.(CropInterface); ok {
            crop.Crop()
          }
        }
      }
    }

    field := db.Statement.Schema.LookUpField("Name")
    // обработка
  }
}

db.Callback().Create().Register("crop_image", cropImage)
// регистрация функции callback для обработки Create
```

### Удаление функций callback

Удаление функции callback из Callbacks

```go
db.Callback().Create().Remove("gorm:create")
// удалить callback `gorm:create` из обработчика Create
```

### Замена функций callback

Заменить callback с идентичным именем на новый

```go
db.Callback().Create().Replace("gorm:create", newCreateFunction)
// заменить callback `gorm:create` новой функцией `newCreateFunction` для обработчика Create
```

### Регистрация функций callback с порядком выполнения

Регистрация функций callback с порядком выполнения

```go
db.Callback().Create().Before("gorm:create").Register("update_created_at", updateCreated)
db.Callback().Create().After("gorm:create").Register("update_created_at", updateCreated)
db.Callback().Query().After("gorm:query").Register("my_plugin:after_query", afterQuery)
db.Callback().Delete().After("gorm:delete").Register("my_plugin:after_delete", afterDelete)
db.Callback().Update().Before("gorm:update").Register("my_plugin:before_update", beforeUpdate)
db.Callback().Create().Before("gorm:create").After("gorm:before_create").Register("my_plugin:before_create", beforeCreate)
```

### Существующие функции callback

GORM предоставляет [некоторые функции callback](https://github.com/go-gorm/gorm/blob/master/callbacks/callbacks.go) для поддержки текущих функций GORM, ознакомьтесь с ними перед началом написания своих плагинов

## Плагин

GORM предоставляет метод `Use` для регистрации плагинов, плагин должен реализовывать интерфейс `Plugin`

```go
type Plugin interface {
  Name() string
  Initialize(*gorm.DB) error
}
```

Метод `Initialize` будет вызван при первом регистрации плагина в GORM, и GORM сохранит его в зарегистрированные плагины, обращайтесь к ним таким образом:

```go
db.Config.Plugins[pluginName]
```

Смотрите [Prometheus](prometheus.html) в качестве примера
