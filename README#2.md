Telegram AI-бот с памятью (Gemini)

Telegram-бот на базе Google Gemini AI с полноценной памятью диалога. Запоминает, о чём вы говорили, и поддерживает контекст.

 Возможности

-Память диалога — бот помнит предыдущие сообщения
- Очистка истории — команда `/clear` сбрасывает диалог
- Быстрые ответы — модель Gemini 1.5 Flash
- Безопасное хранение токенов** — через переменные окружения

Команды

Команда | Описание |

| `/start` | Приветствие и список команд |
| `/help` | Справка |
| `/clear` | Очистить историю диалога |

Установка и запуск

 1. Клонируй репозиторий
bash
git clone https://github.com/coffe-skelet/ai-agent-1
cd ai-agent-1

Установка #2:
pip install pyTelegramBotAPI google-generativeai python-dotenv

Получение ключей:
3. Получи токены

Telegram Bot Token:

    Напиши @BotFather в Telegram

    Создай бота командой /newbot

    Скопируй полученный токен

Gemini API Key:

    Зайди на Google AI Studio

    Нажми «Get API key»

    Скопируй ключ


4. Настрой переменные окружения

Создай файл .env в папке с ботом:
text

TELEGRAM_TOKEN=твой_токен_от_botfather
GEMINI_API_KEY=твой_ключ_gemini

5. Запусти бота
bash

python bot.py

Пример работы
text

Пользователь: Привет, меня зовут Кирилл
Бот: Привет, Кирилл! Чем я могу помочь?

Пользователь: Как меня зовут?
Бот: Тебя зовут Кирилл 😊

Пользователь: /clear
Бот: 🧹 История диалога очищена

Технологии

    Python 3.9+

    pyTelegramBotAPI

    Google Gemini API

    python-dotenv

Контакты

По вопросам и заказам: @gnevovkirill

