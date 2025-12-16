# Favicon файлы

## Инструкция по замене favicon

1. **Сохраните ваше изображение NS** (шестиугольник с буквами NS) на компьютер

2. **Создайте разные размеры favicon** используя онлайн-сервис:
   - Зайдите на https://realfavicongenerator.net/
   - Загрузите ваше изображение
   - Скачайте сгенерированные файлы

3. **Замените placeholder файлы** в папке `static/`:
   - `favicon.ico` - основной favicon (16x16 или 32x32)
   - `favicon-16x16.png` - PNG 16x16 пикселей
   - `favicon-32x32.png` - PNG 32x32 пикселя
   - `apple-touch-icon.png` - для устройств Apple (180x180 пикселей)

4. **Пересоберите сайт**:
   ```bash
   hugo
   ```

## Текущая конфигурация

Favicon уже настроен в `hugo.toml`:

```toml
[params.assets]
  favicon = "favicon.ico"
  favicon16x16 = "favicon-16x16.png"
  favicon32x32 = "favicon-32x32.png"
  apple_touch_icon = "apple-touch-icon.png"
```

Все файлы из папки `static/` автоматически копируются в корень собранного сайта.

