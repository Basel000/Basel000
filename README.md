CREATE TABLE all_messages (
  id SERIAL PRIMARY KEY,
  sender_id BIGINT,
  message_id BIGINT,
  sending_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  content TEXT
);

CREATE TABLE filtered_messages (
  id SERIAL PRIMARY KEY,
  sender_id BIGINT,
  message_id BIGINT,
  sending_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  content TEXT,
  filter_word TEXT
);
package main

import (
    "context"
    "fmt"
    "log"
    "os"
    "strings"

    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
    "github.com/jackc/pgx/v4"
)

var currentFilter string

func main() {
    conn, err := pgx.Connect(context.Background(), fmt.Sprintf("postgres://%s:%s@%s/%s",
        os.Getenv("DB_USER"), os.Getenv("DB_PASSWORD"), os.Getenv("DB_HOST"), os.Getenv("DB_NAME")))
    if err != nil {
        log.Panic(err)
    }
    defer conn.Close(context.Background())

    bot, err := tgbotapi.NewBotAPI(os.Getenv("TELEGRAM_TOKEN"))
    if err != nil {
        log.Panic(err)
    }

    bot.Debug = true

    u := tgbotapi.NewUpdate(0)
    u.Timeout = 60
    updates := bot.GetUpdatesChan(u)

    for update := range updates {
        if update.Message == nil { // ignore any non-Message updates
            continue
        }

        if update.Message.IsCommand() && update.Message.Command() == "filter" {
            currentFilter = update.Message.CommandArguments()
            bot.Send(tgbotapi.NewMessage(update.Message.Chat.ID, "Filter updated to: "+currentFilter))
            continue
        }

        storeMessage(conn, update.Message, currentFilter)
    }
}

func storeMessage(db *pgx.Conn, msg *tgbotapi.Message, filter string) {
    content := msg.Text
    senderID := msg.From.ID
    messageID := msg.MessageID
    sendingDate := msg.Time()

    // Always store to all_messages
    _, err := db.Exec(context.Background(), "INSERT INTO all_messages (sender_id, message_id, sending_date, content) VALUES ($1, $2, $3, $4)",
        senderID, messageID, sendingDate, content)
    if err != nil {
        log.Println("Error storing message:", err)
        return
    }

    // Store to filtered_messages if it matches the filter
    if filter != "" && strings.Contains(strings.ToLower(content), strings.ToLower(filter)) {
        _, err = db.Exec(context.Background(), "INSERT INTO filtered_messages (sender_id, message_id, sending_date, content, filter_word) VALUES ($1, $2, $3, $4, $5)",
            senderID, messageID, sendingDate, content, filter)
        if err != nil {
            log.Println("Error storing filtered message:", err)
        }
    }
}
