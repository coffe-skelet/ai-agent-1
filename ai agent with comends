import imaplib
import email
from email.header import decode_header
import re
import sqlite3
import csv
import json
import os
import getpass
import time
from datetime import datetime

# --- Конфигурация ---
CONFIG_FILE = "email_agent_config.json"
DB_FILE = "leads_database.db"
CSV_FILE = "leads_export.csv"

def setup_database():
    """Создает локальную базу данных SQLite"""
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS leads (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT,
            sender_email TEXT,
            subject TEXT,
            phone TEXT,
            client_email TEXT,
            address TEXT,
            message_preview TEXT,
            processed INTEGER DEFAULT 0
        )
    ''')
    
    conn.commit()
    return conn

def save_to_csv(data):
    """Сохраняет данные в CSV файл (открывается в Excel)"""
    file_exists = os.path.isfile(CSV_FILE)
    
    with open(CSV_FILE, 'a', newline='', encoding='utf-8-sig') as f:
        writer = csv.writer(f)
        
        if not file_exists:
            writer.writerow(['Дата', 'Отправитель', 'Тема', 'Телефон', 'Email клиента', 'Адрес', 'Текст'])
        
        writer.writerow(data)

def setup_credentials():
    """Запрашивает и сохраняет учетные данные"""
    config = {}
    
    if os.path.exists(CONFIG_FILE):
        with open(CONFIG_FILE, 'r', encoding='utf-8') as f:
            config = json.load(f)
        print(f"✅ Загружены сохраненные настройки для {config.get('email', 'неизвестно')}")
    
    if not config.get("email"):
        print("\n" + "="*50)
        print("🚀 ПЕРВЫЙ ЗАПУСК: НАСТРОЙКА EMAIL АГЕНТА")
        print("="*50)
        
        config["email"] = input("📧 Введите ваш Email: ").strip()
        config["password"] = getpass.getpass("🔐 Введите пароль (для Gmail нужен пароль приложения): ").strip()
        
        print("\n📡 Выберите почтовый сервер:")
        print("1. Gmail (imap.gmail.com)")
        print("2. Yandex (imap.yandex.ru)")
        print("3. Mail.ru (imap.mail.ru)")
        print("4. Другой")
        
        choice = input("Ваш выбор (1-4): ").strip()
        
        servers = {
            "1": "imap.gmail.com",
            "2": "imap.yandex.ru",
            "3": "imap.mail.ru"
        }
        
        if choice in servers:
            config["imap_server"] = servers[choice]
        else:
            config["imap_server"] = input("Введите IMAP сервер: ").strip()
        
        config["check_interval"] = 30
        
        with open(CONFIG_FILE, 'w', encoding='utf-8') as f:
            json.dump(config, f, indent=2)
        
        print("✅ Настройки сохранены!")
    
    return config

def extract_data(text):
    """Извлекает контактные данные из текста"""
    result = {
        "phone": None,
        "email": None,
        "address": None
    }
    
    if not text:
        return result
    
    # Поиск телефона (российские форматы)
    phone_patterns = [
        r'(\+7|8)[\s\-]?\(?\d{3}\)?[\s\-]?\d{3}[\s\-]?\d{2}[\s\-]?\d{2}',
        r'\+7\s*\d{3}\s*\d{3}\s*\d{2}\s*\d{2}',
        r'8\s*\d{3}\s*\d{3}\s*\d{2}\s*\d{2}'
    ]
    
    for pattern in phone_patterns:
        match = re.search(pattern, text)
        if match:
            result["phone"] = match.group(0).strip()
            break
    
    # Поиск email
    email_pattern = r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
    emails = re.findall(email_pattern, text)
    if emails:
        result["email"] = emails[0]
    
    # Поиск адреса
    address_patterns = [
        r'(?:Адрес|адрес)[:\s]+([^,\n]+[,\s]*[^,\n]+)',
        r'(?:г\.|город)\s*[А-ЯЁ][а-яё]+\s*,?\s*(?:ул\.|улица|пр\.|проспект|пер\.|переулок)\s*[А-ЯЁа-яё0-9\-\s]+,\s*(?:д\.|дом)\s*\d+[/\d]*[а-я]?',
        r'(?:ул\.|улица|пр\.|проспект)\s*[А-ЯЁа-яё0-9\-\s]+,\s*(?:д\.|дом)\s*\d+'
    ]
    
    for pattern in address_patterns:
        match = re.search(pattern, text, re.IGNORECASE)
        if match:
            if match.groups():
                result["address"] = match.group(1).strip()
            else:
                result["address"] = match.group(0).strip()
            break
    
    return result

def decode_email_content(msg):
    """Декодирует содержимое письма"""
    body = ""
    
    if msg.is_multipart():
        for part in msg.walk():
            content_type = part.get_content_type()
            content_disposition = str(part.get("Content-Disposition"))
            
            if content_type == "text/plain" and "attachment" not in content_disposition:
                try:
                    payload = part.get_payload(decode=True)
                    if payload:
                        body = payload.decode('utf-8', errors='ignore')
                        break
                except:
                    pass
            elif content_type == "text/html" and "attachment" not in content_disposition and not body:
                try:
                    payload = part.get_payload(decode=True)
                    if payload:
                        # Простое удаление HTML тегов
                        html_content = payload.decode('utf-8', errors='ignore')
                        body = re.sub(r'<[^>]+>', ' ', html_content)
                        body = re.sub(r'\s+', ' ', body)
                        break
                except:
                    pass
    else:
        try:
            payload = msg.get_payload(decode=True)
            if payload:
                body = payload.decode('utf-8', errors='ignore')
        except:
            body = str(msg.get_payload())
    
    return body[:1000]  # Ограничиваем длину

def process_emails(mail, conn):
    """Обрабатывает непрочитанные письма"""
    cursor = conn.cursor()
    
    try:
        mail.select("inbox")
        status, messages = mail.search(None, 'UNSEEN')
        
        if status != "OK" or not messages[0]:
            return 0
        
        email_ids = messages[0].split()
        processed_count = 0
        
        for e_id in email_ids:
            try:
                status, msg_data = mail.fetch(e_id, "(RFC822)")
                
                for response_part in msg_data:
                    if isinstance(response_part, tuple):
                        msg = email.message_from_bytes(response_part[1])
                        
                        # Декодируем тему
                        subject = ""
                        subject_header = msg.get("Subject", "")
                        if subject_header:
                            decoded_subject = decode_header(subject_header)
                            for part, encoding in decoded_subject:
                                if isinstance(part, bytes):
                                    subject += part.decode(encoding if encoding else 'utf-8', errors='ignore')
                                else:
                                    subject += part
                        
                        # Отправитель
                        sender = msg.get("From", "Неизвестно")
                        
                        # Тело письма
                        body = decode_email_content(msg)
                        
                        # Извлекаем данные
                        extracted = extract_data(body)
                        
                        # Сохраняем в БД
                        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                        
                        cursor.execute('''
                            INSERT INTO leads 
                            (timestamp, sender_email, subject, phone, client_email, address, message_preview)
                            VALUES (?, ?, ?, ?, ?, ?, ?)
                        ''', (
                            timestamp,
                            sender,
                            subject,
                            extracted["phone"],
                            extracted["email"],
                            extracted["address"],
                            body[:200]
                        ))
                        
                        # Сохраняем в CSV
                        save_to_csv([
                            timestamp,
                            sender,
                            subject,
                            extracted["phone"] or "",
                            extracted["email"] or "",
                            extracted["address"] or "",
                            body[:100]
                        ])
                        
                        print(f"✅ Обработано письмо от: {sender[:50]}")
                        if extracted["phone"] or extracted["email"]:
                            print(f"   📞 Телефон: {extracted['phone'] or 'не найден'}")
                            print(f"   📧 Email: {extracted['email'] or 'не найден'}")
                            print(f"   📍 Адрес: {extracted['address'] or 'не найден'}")
                        
                        processed_count += 1
                        
            except Exception as e:
                print(f"⚠️ Ошибка обработки письма: {e}")
                continue
        
        conn.commit()
        return processed_count
        
    except Exception as e:
        print(f"❌ Ошибка при работе с почтой: {e}")
        return 0

def show_statistics(conn):
    """Показывает статистику работы"""
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM leads")
    total = cursor.fetchone()[0]
    print(f"\n📊 Всего собрано лидов: {total}")
    print(f"📁 Данные сохранены в: {DB_FILE} и {CSV_FILE}")

def main():
    print("""
    ╔═══════════════════════════════════════╗
    ║     🤖 EMAIL АГЕНТ-АВТОМАТИЗАТОР     ║
    ║   Сбор контактов из входящих писем   ║
    ╚═══════════════════════════════════════╝
    """)
    
    # Настройка
    config = setup_credentials()
    conn = setup_database()
    
    print(f"\n🔄 Подключение к {config['imap_server']}...")
    
    try:
        # Подключение к почте
        mail = imaplib.IMAP4_SSL(config["imap_server"])
        mail.login(config["email"], config["password"])
        print(f"✅ Успешное подключение к почте {config['email']}")
        
        print(f"\n🔍 Мониторинг входящих писем каждые {config['check_interval']} секунд...")
        print("Для остановки нажмите Ctrl+C\n")
        
        while True:
            try:
                processed = process_emails(mail, conn)
                
                if processed > 0:
                    show_statistics(conn)
                
                # Показываем что агент жив
                print(f"⏳ Ожидание... {datetime.now().strftime('%H:%M:%S')}", end='\r')
                
                time.sleep(config["check_interval"])
                
            except KeyboardInterrupt:
                break
            except Exception as e:
                print(f"\n⚠️ Временная ошибка: {e}")
                time.sleep(10)
                
    except KeyboardInterrupt:
        print("\n\n👋 Агент остановлен пользователем")
    except Exception as e:
        print(f"\n❌ Критическая ошибка: {e}")
        print("\n💡 Проверьте:")
        print("1. Правильность пароля (для Gmail нужен пароль приложения)")
        print("2. Включен ли IMAP в настройках почты")
        print("3. Правильность IMAP сервера")
    finally:
        show_statistics(conn)
        conn.close()
        try:
            mail.logout()
        except:
            pass
        print("\n✨ Работа завершена!")

if __name__ == "__main__":
    main()
