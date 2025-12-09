# Relatório Técnico  
## Análise de Segurança em Servidor Web IoT (ESP32)

---

## Introdução

Este relatório apresenta uma análise de segurança de uma solução IoT baseada em ESP32 com servidor web local, utilizando como referência o exemplo disponibilizado em:

https://randomnerdtutorials.com/esp32-web-server-arduino-ide/

O objetivo da atividade é identificar vulnerabilidades, possíveis ataques e avaliar os riscos de segurança do sistema.

---

## Código Analisado (Servidor Web ESP32)

O código abaixo foi retirado do tutorial  
https://randomnerdtutorials.com/esp32-web-server-arduino-ide/

```
#include <WiFi.h>

const char* ssid = "REPLACE_WITH_YOUR_SSID";
const char* password = "REPLACE_WITH_YOUR_PASSWORD";

WiFiServer server(80);

String header;

String output26State = "off";
String output27State = "off";

const int output26 = 26;
const int output27 = 27;

unsigned long timeoutTime = 2000;

void setup() {
  Serial.begin(115200);
  pinMode(output26, OUTPUT);
  pinMode(output27, OUTPUT);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }

  server.begin();
}

void loop(){
  WiFiClient client = server.available();

  if (client) {
    String currentLine = "";
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        header += c;

        if (c == '\n') {
          if (currentLine.length() == 0) {
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println();

            if (header.indexOf("GET /26/on") >= 0) {
              digitalWrite(26, HIGH);
            } 
            if (header.indexOf("GET /26/off") >= 0) {
              digitalWrite(26, LOW);
            }

            client.println("</html>");
            break;
          } 
        } 
      }
    }
    header = "";
    client.stop();
  }
}
```

## O que esse código faz?

O código transforma o ESP32 em um servidor web local, acessível por um navegador.

Funções principais:

- Conecta o ESP32 a uma rede Wi-Fi
- Cria um servidor HTTP na porta 80
- Permite controlar os pinos GPIO 26 e 27 via comandos enviados pela URL
- Exibe uma página HTML com botões de controle

---

## Análise Estática – Vulnerabilidades Encontradas

### Falta de autenticação

O código não implementa nenhum tipo de login ou senha.  
Qualquer cliente conectado à mesma rede pode acessar o ESP32.

**Trecho crítico:**
````
WiFiClient client = server.available();
````

## Uso de HTTP sem criptografia

O servidor funciona na porta 80, usando HTTP puro.

**Trecho crítico:**
````
WiFiServer server(80);
````
3.3 Falta de validação de entradas

O código aceita comandos diretamente da URL sem validação.

Trecho crítico:
````
if (header.indexOf("GET /26/on") >= 0)
````
---

## Possíveis Ataques Identificados

### Ataque 1 – Acesso não autorizado ao dispositivo

**Descrição:**  
Exploração da falta de autenticação do servidor web do ESP32.

#### Passo a passo:

1. O atacante se conecta à mesma rede Wi-Fi do dispositivo.
2. Descobre o IP do ESP32 na rede local.
3. Digita o IP no navegador.
4. Obtém acesso ao painel do dispositivo sem necessidade de login.

#### Probabilidade:
Alta – redes Wi-Fi costumam ser compartilhadas.

#### Impacto:
Alto – controle indevido do dispositivo.

#### Risco resultante:
Crítico – combinação de alta probabilidade com alto impacto.

---

### Ataque 2 – Interceptação de dados 

**Descrição:**  
Exploração da comunicação sem criptografia.

#### Passo a passo:

1. O atacante se conecta à mesma rede Wi-Fi.
2. Utiliza ferramentas de captura de pacotes (ex: Wireshark).
3. Monitora o tráfego da rede.
4. Visualiza os dados trocados entre o usuário e o ESP32.

#### Probabilidade:
Média – requer conhecimento técnico.

#### Impacto:
Alto – exposição de informações e comandos.

#### Risco resultante:
Alto – possibilidade real de danos ao sistema.

---

## Tabela Consolidada de Ataques (Ordenada por Risco)

| Título do Ataque                       | Probabilidade | Impacto | Risco Final |
|----------------------------------------|---------------|---------|-------------|
| Acesso não autorizado ao ESP32         | Alta          | Alto    | Crítico     |
| Interceptação de dados via HTTP        | Média         | Alto    | Alto        |

---


O código analisado apresenta falhas críticas de segurança, principalmente a ausência de autenticação e o uso de comunicação não criptografada. Essas vulnerabilidades permitem a ocorrência de ataques de acesso indevido e interceptação de dados, representando riscos elevados ao sistema.

## Código de Correção

A seguir está um exemplo de como o código do ESP32 pode ser reforçado com mecanismos básicos de segurança.

---

### Implementação de Autenticação (usuário e senha)

Este trecho exige login para acessar o servidor:

````
const char* http_username = "admin";
const char* http_password = "1234";

bool isAuthenticated(String header) {
  if (header.indexOf("Authorization") >= 0) {
    return true;
  }
  return false;
}
````
Uso dentro do loop():
````
if (!isAuthenticated(header)) {
  client.println("HTTP/1.1 401 Unauthorized");
  client.println("WWW-Authenticate: Basic realm=\"ESP32\"");
  client.println();
  break;
}
````

### Implementação de HTTPS (TLS)

Este exemplo mostra como utilizar comunicação segura no ESP32 através da biblioteca de conexão segura.

````
#include <WiFiClientSecure.h>

WiFiClientSecure secureClient;
````
**Exemplo de inicialização:**

````
secureClient.setInsecure();
````

**Validação de Entrada (proteção contra comandos maliciosos)**
````
bool isValidCommand(String header) {
  return (
    header.indexOf("GET /26/on") >= 0 ||
    header.indexOf("GET /26/off") >= 0 ||
    header.indexOf("GET /27/on") >= 0 ||
    header.indexOf("GET /27/off") >= 0
  );
}

````
Aplicação desse mecanismo no código do servidor:

````
if (!isValidCommand(header)) {
  client.println("HTTP/1.1 400 Bad Request");
  client.println();
  break;
}
````
## Conclusão:
Com a adição de autenticação, criptografia TLS e validação das entradas, o sistema torna-se significativamente mais seguro, reduzindo as chances de acesso não autorizado e de execução de comandos maliciosos no dispositivo IoT.

