Ansible playbooks для автоматизации установки и обновления платформы 1С:Предприятие на Windows-хостах через WSL2.

## 📋 Содержание
- [Возможности](#-возможности)
- [Архитектура решения](#-архитектура-решения)
- [Установка WSL2](#-установка-wsl2)
- [Настройка Ansible в WSL2](#-настройка-ansible-в-wsl2)
- [Настройка Windows хостов](#26)
- [Создание инвентаря](#-создание-инвентаря)
- [Плейбуки](#-плейбуки)
- [Проверка работы](#-проверка-работы)
- [Установка 1С из архива](#-установка-1с-из-архива)
- [Полезные ссылки](#-полезные-ссылки)
- [Лицензия](#-лицензия)

## 🚀 Возможности

- Полностью автоматизированная установка 1С на Windows хосты
- Работа через WSL2 (Windows Subsystem for Linux)
- Поддержка WinRM для подключения к Windows
- Копирование и распаковка архивов на целевых машинах
- Установка с тихими параметрами (/S /quiet /norestart)
- Проверка успешности установки через WMI и реестр
- Очистка временных файлов после установки

## Установка WSL2

Откройте **PowerShell от имени Администратора** и выполните:
```
wsl --install
```
Важно! После установки перезагрузите компьютер
 
## 🔧 Установка ansible

Обновление пакетов
```
sudo apt update
```
Установка Ansible
```
sudo apt install ansible -y
```
Проверка установки
```
ansible --version
```
## 🚀 Настройка WinRM

Установка pip и pywinrm
```
sudo apt install python3-pip -y
pip3 install pywinrm
```

Или через системные пакеты
```
sudo apt install python3-winrm -y
```
## 🔧 Настройка WinRM

Этап 1: Подготовка
Запускаем PowerShell от имени Администратора. Сбросим настройки, чтобы избежать конфликтов со старыми конфигами.
# Включение WinRM
```
Enable-PSRemoting -Force
```
# Создание пользователя для Ansible
```
# 4.1 УДАЛЯЕМ СТАРОГО ПОЛЬЗОВАТЕЛЯ (Если вы уже пробовали создать ранее и хотите пересоздать)
# Remove-LocalUser -Name "AnsibleUser" -ErrorAction SilentlyContinue

# 4.2 СОЗДАЕМ НОВОГО ПОЛЬЗОВАТЕЛЯ
Write-Host "`nСоздаем пользователя AnsibleUser..." -ForegroundColor Cyan
$Password = ConvertTo-SecureString "n0TiniTed0" -AsPlainText -Force
New-LocalUser -Name "AnsibleUser" -Password $Password -Description "Ansible automation user" -AccountNeverExpires -PasswordNeverExpires

# 4.3 ДОБАВЛЯЕМ В ГРУППЫ (Критически важно!)
Write-Host "Добавляем пользователя в группы..." -ForegroundColor Cyan

# Группа администраторов (нужна для управления системой)
Add-LocalGroupMember -Group "Администраторы" -Member "AnsibleUser" -ErrorAction SilentlyContinue
# Группа удаленного управления (Remote Management Users)
Add-LocalGroupMember -Group "Пользователи удаленного управления" -Member "AnsibleUser" -ErrorAction SilentlyContinue

# Проверяем, что пользователь добавлен
Write-Host "`nПроверка членства в группах:" -ForegroundColor Yellow
Get-LocalGroupMember -Group "Администраторы" | Where-Object {$_.Name -like "*AnsibleUser*"}
Get-LocalGroupMember -Group "Пользователи удаленного управления" | Where-Object {$_.Name -like "*AnsibleUser*"}
```
Этап 2: Настройка брандмауэра и прав доступа
```
# 3.1 НАСТРОЙКА БРАНДМАУЭРА
Write-Host "`nНастраиваем брандмауэр..." -ForegroundColor Cyan

# Закрываем старый HTTP порт (5985) для надежности
Remove-NetFirewallRule -Name "WinRM HTTP" -ErrorAction SilentlyContinue

# Открываем новый HTTPS порт (5986)
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Name "WinRM HTTPS" -Profile Any -LocalPort 5986 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue

# Проверяем, что правило создалось и включено
Get-NetFirewallRule -DisplayName "WinRM HTTPS" | Format-Table Name, Enabled, Direction, Action

# 3.2 НАСТРОЙКА АУТЕНТИФИКАЦИИ (Важно!)
Write-Host "`nНастраиваем параметры аутентификации WinRM..." -ForegroundColor Cyan

# Запрещаем незашифрованные соединения (только HTTPS)
winrm set winrm/config/service '@{AllowUnencrypted="false"}'

# Включаем Basic аутентификацию (часто используется Ansible)
winrm set winrm/config/service/auth '@{Basic="true"}'

# Включаем Negotiate/NTLM (для совместимости)
winrm set winrm/config/service/auth '@{Negotiate="true"}'
winrm set winrm/config/service/auth '@{Kerberos="true"}'

# 3.3 ПЕРЕЗАПУСКАЕМ СЛУЖБУ ДЛЯ ПРИМЕНЕНИЯ ВСЕХ НАСТРОЕК
Restart-Service WinRM
Write-Host "Служба WinRM перезапущена." -ForegroundColor Green
```
Этап 3: Создание сертификата и HTTPS-слушателя
```
# 2.1 СОЗДАЕМ СЕРТИФИКАТ
Write-Host "Создаем самоподписанный сертификат..." -ForegroundColor Cyan
$cert = New-SelfSignedCertificate -DnsName $env:COMPUTERNAME -CertStoreLocation "Cert:\LocalMachine\My" -KeyLength 2048 -KeyAlgorithm "RSA" -HashAlgorithm "SHA256"

# Сохраняем отпечаток (Thumbprint) в переменную, он нам пригодится
$thumbprint = $cert.Thumbprint

Write-Host "Сертификат создан:" -ForegroundColor Green
Write-Host "  Thumbprint (отпечаток): $thumbprint" -ForegroundColor White
Write-Host "  Subject: $($cert.Subject)"
Write-Host "  Истекает: $($cert.NotAfter)"

# 2.2 СОЗДАЕМ HTTPS СЛУШАТЕЛЬ
Write-Host "`nСоздаем HTTPS слушатель на порту 5986..." -ForegroundColor Cyan
winrm create winrm/config/listener?Address=*+Transport=HTTPS "@{Hostname=`"$env:COMPUTERNAME`"; CertificateThumbprint=`"$thumbprint`"}"

# 2.3 ПРОВЕРЯЕМ, ЧТО СЛУШАТЕЛЬ ПОЯВИЛСЯ
Write-Host "`nСписок активных слушателей WinRM:" -ForegroundColor Yellow
winrm enumerate winrm/config/listener
```
Полезные ссылки!
```
https://docs.ansible.com/projects/ansible/latest/os_guide/windows_winrm.html
```
```
https://gist.github.com/nikhilsingnurkar/9776116d44446a3f5da64d71cfafe57f#file-configureremotingforansible-ps1
```


Ping
```
ansible -i hosts.ini windows_depo_new_pc -m win_ping
```
