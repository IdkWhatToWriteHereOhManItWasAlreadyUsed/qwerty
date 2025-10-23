Конечно! Если ты уже **скопировал `.sys`-файл драйвера на виртуальную машину (ВМ) в VirtualBox**, и хочешь **просто запустить его без сложных настроек**, вот пошаговая инструкция — **минимум действий, максимум результата**.

> ⚠️ Это работает **только если драйвер не требует специального оборудования** (т.е. это виртуальный/программный драйвер, как в твоём случае).

---

## 📌 Условия
1. У тебя есть:
   - `MyDriver.sys` — сам драйвер
   - `MyDriver.inf` — файл установки (обязательно!)
2. На ВМ включён **Test Signing Mode**:
   ```cmd
   bcdedit /set testsigning on
   ```
   → **перезагрузи ВМ** после этого.
3. Ты вошёл в систему **от администратора**.

---

## ✅ Шаг 1: Подготовь `.inf` файл (если его нет)

Если у тебя **только `.sys`**, но нет `.inf` — создай простой `.inf` рядом с ним.

Пример `MyDriver.inf`:

```ini
[Version]
Signature="$WINDOWS NT$"
Class=System
ClassGuid={4d36e97d-e325-11ce-bfc1-08002be10318}
Provider=%ManufacturerName%
DriverVer=10/23/2025,1.0.0.0

[Manufacturer]
%ManufacturerName%=Standard,NT$ARCH$

[Standard.NT$ARCH$]
%DeviceName%=MyDriver_Install, Root\MyDriver

[MyDriver_Install]
CopyFiles=DriverFiles

[MyDriver_Install.Services]
AddService=MyDriver,2,MyDriver_Service_Inst

[MyDriver_Service_Inst]
ServiceType=1               ; KERNEL_DRIVER
StartType=3                 ; DEMAND_START (запуск вручную)
ErrorControl=1              ; NORMAL
ServiceBinary=%12%\MyDriver.sys
DisplayName=%ServiceName%

[DriverFiles]
MyDriver.sys

[Strings]
ManufacturerName="MyLab"
DeviceName="My Virtual Driver"
ServiceName="MyDriver"
```

Сохрани как `MyDriver.inf` в той же папке, что и `MyDriver.sys`.

---

## ✅ Шаг 2: Установи драйвер через `pnputil`

Открой **Командную строку от имени Администратора** на ВМ:

```cmd
cd C:\путь\к\папке\с\драйвером
pnputil /add-driver MyDriver.inf /install
```

✅ Если всё хорошо — увидишь:
```
Driver package added successfully.
Published Name: oemXX.inf
```

> 💡 `pnputil` автоматически скопирует `.sys` в `C:\Windows\System32\drivers\`.

---

## ✅ Шаг 3: Запусти драйвер как службу

Драйверы режима ядра в Windows управляются через **диспетчер служб**.

Выполни:

```cmd
sc create MyDriver binPath="system32\drivers\MyDriver.sys" type=kernel
sc start MyDriver
```

> 🔹 `MyDriver` — имя должно совпадать с `ServiceName` в `.inf`.

Проверь статус:

```cmd
sc query MyDriver
```

Если статус **RUNNING** — драйвер работает!

---

## ✅ Шаг 4: Проверь, что драйвер "ожил"

### Вариант A: Используй **DebugView**
1. Скачай [DebugView](https://learn.microsoft.com/en-us/sysinternals/downloads/debugview)
2. Запусти **от администратора**
3. Включи:
   - **Capture → Kernel Output**
4. Если в драйвере есть `DbgPrint("Hello\n");` — ты это увидишь.

### Вариант B: Проверь через диспетчер задач
- Открой **Диспетчер задач → Службы**
- Найди `MyDriver` — статус должен быть **Работает**

---

## 🛑 Как остановить и удалить драйвер

```cmd
sc stop MyDriver
sc delete MyDriver
```

> 💡 После `sc delete` драйвер **не удаляется из системы полностью** — только запись службы.  
> Чтобы полностью удалить:  
> ```cmd
> pnputil /delete-driver oemXX.inf /force
> ```
> (где `oemXX.inf` — имя из шага 2)

---

## ❗ Возможные ошибки и решения

| Ошибка | Причина | Решение |
|-------|--------|--------|
| `The service binary does not exist` | Нет `.sys` в `System32\drivers` | Убедись, что `pnputil` выполнился успешно |
| `Access is denied` | Не от администратора | Запусти CMD от админа |
| `Test mode is not enabled` | Не включён Test Signing | Выполни `bcdedit /set testsigning on` + перезагрузка |
| `Driver failed to load` | Ошибка в коде драйвера | Смотри через DebugView или Event Viewer → System |

---

## 🎯 Пример: полный цикл

```cmd
# 1. Включи тестовый режим (один раз)
bcdedit /set testsigning on
# → перезагрузи ВМ

# 2. Установи драйвер
pnputil /add-driver MyDriver.inf /install

# 3. Зарегистрируй службу
sc create MyDriver binPath="system32\drivers\MyDriver.sys" type=kernel

# 4. Запусти
sc start MyDriver

# 5. Посмотри лог
# → запусти DebugView и смотри Kernel Output

# 6. Останови
sc stop MyDriver
sc delete MyDriver
```

---

Готово! Теперь ты можешь **быстро тестировать любой `.sys`-драйвер** на ВМ в VirtualBox.

Если хочешь — могу дать **готовый архив с шаблоном драйвера + `.inf` + скриптом запуска**.

Удачи! 💻🔧
