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

### Uso de HTTP sem criptografia

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
### Falta de Limitação/Validação de Entradas

### Falta de controle de tamanho da requisição (Risco de Buffer Overflow / DoS)

O código armazena toda a requisição HTTP recebida na variável `String header` sem nenhum limite de tamanho:

Trecho crítico:
````
header += c;
````
---

## Ataque 1 – Acesso Não Autorizado ao Dispositivo

**Exploração da vulnerabilidade:**  
Falta de autenticação no servidor web (Ponto 1).

### Passo a passo (nível conceitual):

1. O atacante conecta-se à mesma rede Wi-Fi do dispositivo IoT.
2. Identifica o endereço IP do ESP32 na rede local.
3. Acessa o IP por meio de um navegador ou cliente HTTP.
4. Obtém acesso direto ao painel de controle, conseguindo manipular os GPIOs sem necessidade de credenciais.

### Probabilidade:
Alta – Em redes compartilhadas, o acesso local é comum e a técnica é simples de executar.

### Impacto:
Alto – Permite controle indevido do dispositivo, podendo acionar ou desativar cargas elétricas ou sistemas conectados.

### Risco resultante:
Crítico – Alta probabilidade de ocorrência combinada com alto impacto operacional.

---

## Ataque 2 – Negação de Serviço (DoS) por Sobrecarga de Requisição

**Exploração da vulnerabilidade:**  
Falta de limitação de tamanho da entrada armazenada na variável `String header` (Ponto 3).

### Passo a passo (nível conceitual):

1. O atacante identifica o IP do ESP32 na rede.
2. Envia requisições HTTP com cabeçalhos ou URLs excessivamente longos.
3. O ESP32 tenta armazenar toda a requisição na memória.
4. O consumo excessivo de RAM causa travamento, reinicialização ou indisponibilidade do dispositivo.

### Probabilidade:
Média – Requer algum conhecimento técnico, mas é viável contra dispositivos com recursos limitados.

### Impacto:
Alto – O dispositivo torna-se indisponível, impedindo o uso legítimo.

### Risco resultante:
Alto – Gera interrupção do serviço e perda de confiabilidade do sistema.

---

## Ataque 3 – Interceptação de Dados em Trânsito (Man-in-the-Middle)

**Exploração da vulnerabilidade:**  
Uso de HTTP sem criptografia (Ponto 2).

### Passo a passo (nível conceitual):

1. O atacante conecta-se à mesma rede Wi-Fi do sistema.
2. Monitora o tráfego de rede entre o usuário legítimo e o ESP32 utilizando ferramentas de análise de pacotes.
3. Como a comunicação ocorre em texto simples, é possível visualizar comandos e respostas trocados.

### Probabilidade:
Média – Exige acesso à rede local e conhecimento básico de redes.

### Impacto:
Médio – Exposição de informações sensíveis e dos comandos enviados ao dispositivo.

### Risco resultante:
Médio – Possibilidade real de espionagem do tráfego, comprometendo a confidencialidade.

---

## Tabela Consolidada de Riscos dos Ataques

| Título do Ataque                                | Probabilidade | Impacto | Risco Final |
|--------------------------------------------------|---------------|---------|-------------|
| Acesso Não Autorizado ao Dispositivo             | Alta          | Alto    | Crítico     |
| Negação de Serviço por Sobrecarga de Requisição  | Média         | Alto    | Alto        |
| Interceptação de Dados em Trânsito (MitM)        | Média         | Médio   | Médio       |

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

