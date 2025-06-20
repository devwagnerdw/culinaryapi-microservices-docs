# 🍽️ CulinaryAPI - Arquitetura de Microsserviços

Este projeto é uma plataforma de gerenciamento de pedidos para um restaurante, construída com base em uma arquitetura de **microsserviços assíncronos**. O sistema é composto por 8 microsserviços independentes, com comunicação via **mensageria orientada a eventos** (Event-Driven Architecture) e centralização de configurações por meio de um repositório dedicado.

---
## 🧭 Visão Geral da Arquitetura
![Captura de tela 2025-06-20 111410](https://github.com/user-attachments/assets/e0647c37-0880-4c4c-879d-e669d9324119)

## 📦 Microsserviços

| Serviço                           | Descrição                                                                 |
|-----------------------------------|---------------------------------------------------------------------------|
| [`culinaryapi-auth`](https://github.com/devwagnerdw/culinaryapi-auth)              | Responsável por autenticação e cadastro de usuários e entregadores.       |
| [`culinaryapi-user-Service`](https://github.com/devwagnerdw/culinaryapi-UserService)       | Gerencia os dados do cliente e seus endereços.                            |
| [`culinaryapi-Menu-Service`](https://github.com/devwagnerdw/culinaryapi-MenuService)       | Gerencia o cadastro de pratos e itens do cardápio.                        |
| [`culinaryapi-Order-Service`](https://github.com/devwagnerdw/culinaryapi-Order-Service)                  | Responsável pela criação e controle dos pedidos.                          |
| [`culinaryapi-Delivery-Service`](https://github.com/devwagnerdw/culinaryapi-Delivery-Service)   | Gerencia os entregadores e o fluxo de entrega.                            |
| [`culinaryapi-gateway`](https://github.com/devwagnerdw/culinaryapi-gateway)           | Roteador de requisições HTTP. Entrada única para o sistema.               |
| [`service-registry`](https://github.com/devwagnerdw/culinaryapi-service-registry)               | Registro e descoberta de serviços (Eureka).                               |
| [`config-server`](https://github.com/devwagnerdw/culinaryapi-config-server)                  | Servidor de configuração centralizado (Spring Cloud Config Server).       |
| [`culinaryapi-config-server-repo`](https://github.com/devwagnerdw/culinaryapi-config-server-repo)| Repositório Git que armazena os `application.yaml` de todos os serviços. |

---

### 1. **Cadastro de Cliente ou Entregador**
- A requisição é enviada via **Gateway** para o `culinaryapi-auth`.
- Esse serviço é responsável por gerenciar **todo o fluxo de autenticação** da arquitetura, incluindo login, cadastro e validação de credenciais.
- Ao receber os dados, o serviço salva no banco local.
- Em seguida:
  - **Se for um cliente comum**, o serviço publica o evento **`UserEvent`**.
    - Esse evento é consumido por:
      - `culinaryapi-user-Service`
      - `Order-Service`
  - **Se for um entregador**, o serviço publica o evento **`DeliverymanEvent`**.
    - Esse evento é consumido **somente** pelo:
      - `culinaryapi-Delivery-Service`
---

### 2. **Cadastro de Endereço**
- O cliente acessa o `culinaryapi-user-Service` para cadastrar seu endereço.
- Ao concluir, o serviço emite o evento **`UserServiceEvent`**.
- Esse evento é consumido pelo `Order-Service`, que associa o endereço ao cliente.

---

### 3. **Cadastro de Itens do Cardápio**
- Realizado no `culinaryapi-Menu-Service`.
- Cada novo item gera um evento **`MenuServiceEvent`**.
- O `Order-Service` consome esse evento para manter o catálogo de opções atualizado.

---

### 4. **Criação de Pedido**
- O cliente envia um pedido ao `Order-Service`.
- O pedido entra inicialmente com o status **`PENDING`**, aguardando ação do restaurante.
- O restaurante pode então atualizar o status na seguinte sequência:
  - **`CONFIRMED`** – pedido aceito pelo restaurante.
  - **`PREPARING`** – o preparo do pedido foi iniciado.
  - **`READY_FOR_PICKUP`** – pedido pronto para ser retirado por um entregador.
- Somente após o status ser alterado para **`READY_FOR_PICKUP`**, o `Order-Service` publica um evento **`OrderEvent`**, que é consumido pelo `culinaryapi-Delivery-Service`.

---

### 5. **Cadastro de Entregador**
- O entregador se cadastra via `culinaryapi-auth`.
- É publicado o evento **`DeliverymanEvent`**, consumido apenas pelo `culinaryapi-Delivery-Service`.

---

### 6. **Cadastro de Veículo**
- O entregador cadastra um veículo no `culinaryapi-Delivery-Service`.
- Apenas entregadores com veículo cadastrado podem retirar pedidos.

---

### 7. **Entrega do Pedido**
- Ao coletar o pedido, o entregador atualiza o status para `OUT_FOR_DELIVERY`.
- Quando o pedido chega ao cliente, é necessário inserir um **código de segurança**:
  - Os **4 últimos dígitos do número de celular cadastrado**.
- Se o código estiver correto:
  - O status é alterado para `DELIVERED`.
  - Um evento é enviado à fila, informando `culinaryapi-Delivery-Service` e `Order-Service`.
- Se o código estiver incorreto:
  - Uma mensagem de erro é exibida.
  - **Nenhuma ação automática é tomada** (o pedido não é mais cancelado).

---

## 📬 Filas e Eventos

| Evento/Fila              | Produzido por              | Consumido por                                       |
|--------------------------|----------------------------|------------------------------------------------------|
| `UserEvent`              | `culinaryapi-auth`         | `culinaryapi-user-Service`, `Order-Service`          |
| `UserServiceEvent`       | `culinaryapi-user-Service` | `culinaryapi-Order-Service`                                      |
| `MenuServiceEvent`       | `culinaryapi-Menu-Service` | `culinaryapi-Order-Service`                                      |
| `OrderEvent`             | `culinaryapi-Order-Service`            | `culinaryapi-Delivery-Service`                       |
| `DeliverymanEvent`       | `culinaryapi-auth`         | `culinaryapi-Delivery-Service`                       |
| `DeliveryServiceEvent`   | `culinaryapi-Delivery-Service` | `culinaryapi-Order-Service`                                  |

---

## 🧠 Regras de Negócio

- **Cadastro de Endereço** é obrigatório antes de realizar um pedido.
- **Cadastro de Veículo** é obrigatório para entregadores aceitarem pedidos.
- **Código de segurança de 4 dígitos** é necessário para finalizar entregas.
- O sistema apenas exibe erro quando o código estiver incorreto — sem cancelar o pedido automaticamente.
- Todos os status do pedido são propagados entre serviços por **eventos assíncronos**.

---

## 🛠️ Tecnologias e Padrões Utilizados

- Java com Spring Boot 3+
- Spring Cloud (Gateway, Eureka, Config Server)
- RabbitMQ (mensageria)
- Comunicação assíncrona baseada em eventos (event-driven)
- spring security
- Autenticação com JWT

---

## 🎯 Objetivos da Arquitetura

- **Desacoplamento**: Cada serviço é independente e especializado.
- **Escalabilidade**: Escala horizontal com facilidade por serviço.
- **Resiliência**: Tolerância a falhas por meio de filas assíncronas.
- **Auditoria e rastreabilidade** via eventos.
