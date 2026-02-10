# client.go


package main

import (
 "bufio"
 "fmt"
 "net"
 "os"
 "strings"
)

func main() {
 fmt.Println("Запуск клиента чата...")
 
 conn, err := net.Dial("tcp", "localhost:8080")
 if err != nil {
  fmt.Println("Ошибка подключения к серверу:", err)
  return
 }
 defer conn.Close()
 
 fmt.Println("Подключено к серверу")
 
 // Канал для сообщений с емкостью 5
 messageChan := make(chan string, 5)
 
 // Запускаем горутину для вывода сообщений
 go outputMessages(messageChan)
 
 // Запускаем горутину для чтения сообщений от сервера
 go readServerMessages(conn, messageChan)
 
 // Читаем ввод пользователя и отправляем на сервер
 reader := bufio.NewReader(os.Stdin)
 
 for {
  fmt.Print("Вы: ")
  text, err := reader.ReadString('\n')
  if err != nil {
   fmt.Println("Ошибка чтения ввода:", err)
   break
  }
  
  text = strings.TrimSpace(text)
  if text == "" {
   continue
  }
  
  _, err = conn.Write([]byte(text + "\n"))
  if err != nil {
   fmt.Println("Ошибка отправки сообщения:", err)
   break
  }
 }
 
 fmt.Println("Клиент завершает работу")
}

func readServerMessages(conn net.Conn, messageChan chan<- string) {
 reader := bufio.NewReader(conn)
 
 for {
  message, err := reader.ReadString('\n')
  if err != nil {
   messageChan <- "Соединение с сервером потеряно"
   close(messageChan)
   return
  }
  
  message = strings.TrimSpace(message)
  messageChan <- message
 }
}

func outputMessages(messageChan <-chan string) {
 for {
  select {
  case msg, ok := <-messageChan:
   if !ok {
    fmt.Println("Канал сообщений закрыт")
    return
   }
   fmt.Println("\n" + msg)
   fmt.Print("Вы: ")
  }
 }
}


# server.go

package main

import (
 "bufio"
 "fmt"
 "net"
 "os"
 "strings"
 "sync"
)

type Client struct {
 conn net.Conn
 name string
}

var (
 clients     = make(map[*Client]bool)
 clientsLock sync.RWMutex
 messageChan = make(chan string, 10)
)

func main() {
 fmt.Println("Запуск сервера чата...")
 
 listener, err := net.Listen("tcp", ":8080")
 if err != nil {
  fmt.Println("Ошибка запуска сервера:", err)
  return
 }
 defer listener.Close()
 
 fmt.Println("Сервер слушает на порту 8080")
 
 // Запускаем горутину для вывода сообщений
 go outputMessages()
 
 // Запускаем горутину для ввода сообщений от сервера
 go serverInput()
 
 for {
  conn, err := listener.Accept()
  if err != nil {
   fmt.Println("Ошибка подключения:", err)
   continue
  }
  
  go handleClient(conn)
 }
}

func handleClient(conn net.Conn) {
 defer conn.Close()
 
 // Запрашиваем имя пользователя
 conn.Write([]byte("Введите ваше имя: "))
 
 reader := bufio.NewReader(conn)
 name, err := reader.ReadString('\n')
 if err != nil {
  fmt.Println("Ошибка чтения имени:", err)
  return
 }
 
 name = strings.TrimSpace(name)
 client := &Client{conn: conn, name: name}
 
 // Добавляем клиента
 clientsLock.Lock()
 clients[client] = true
 clientsLock.Unlock()
 
 // Уведомляем всех о новом пользователе
 messageChan <- fmt.Sprintf("Пользователь %s присоединился к чату", name)
 broadcast(fmt.Sprintf("Пользователь %s присоединился к чату", name))
 
 fmt.Printf("Новое подключение: %s (%s)\n", name, conn.RemoteAddr())
 
 // Обрабатываем сообщения от клиента
 for {
  message, err := reader.ReadString('\n')
  if err != nil {
   break
  }
  
  message = strings.TrimSpace(message)
  if message == "" {
   continue
  }
  
  fullMessage := fmt.Sprintf("%s: %s", name, message)
  messageChan <- fullMessage
  broadcast(fullMessage)
 }
 
 // Удаляем клиента при отключении
 clientsLock.Lock()
 delete(clients, client)
 clientsLock.Unlock()
 
 leaveMessage := fmt.Sprintf("Пользователь %s покинул чат", name)
 messageChan <- leaveMessage
 broadcast(leaveMessage)
 
 fmt.Printf("Отключение: %s (%s)\n", name, conn.RemoteAddr())
}

func broadcast(message string) {
 clientsLock.RLock()
 defer clientsLock.RUnlock()
 
 for client := range clients {
  _, err := client.conn.Write([]byte(message + "\n"))
  if err != nil {
   // Не удалось отправить - клиент возможно отключился
   continue
  }
 }
}

func outputMessages() {
 for {
  select {
  case msg := <-messageChan:
   fmt.Println(msg)
  }
 }
}

func serverInput() {
 reader := bufio.NewReader(os.Stdin)
 
 for {
  fmt.Print("Сервер: ")
  text, err := reader.ReadString('\n')
  if err != nil {
   fmt.Println("Ошибка чтения ввода:", err)
   continue
  }
  
  text = strings.TrimSpace(text)
  if text == "" {
   continue
  }
  
  message := fmt.Sprintf("Сервер: %s", text)
  messageChan <- message
  broadcast(message)
 }
}
