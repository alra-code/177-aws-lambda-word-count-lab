<div align="center">

![header](https://capsule-render.vercel.app/api?type=waving&color=232F3E&height=200&section=header&text=Lambda%20Word%20Count&fontSize=46&fontColor=FFFFFF&animation=fadeIn&desc=S3%20%E2%86%92%20Lambda%20%E2%86%92%20SNS%3A%20arquitetura%20serverless%20orientada%20a%20eventos&descSize=17&descAlignY=72)

![Lambda](https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white)
![S3](https://img.shields.io/badge/Amazon-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![SNS](https://img.shields.io/badge/Amazon-SNS-D6336C?style=for-the-badge&logo=amazonaws&logoColor=white)
![CloudWatch](https://img.shields.io/badge/CloudWatch-E7157B?style=for-the-badge&logo=amazonaws&logoColor=white)
![IAM](https://img.shields.io/badge/AWS-IAM-DD344C?style=for-the-badge&logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Serverless](https://img.shields.io/badge/Serverless-Event--driven-8C4FFF?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-2ea44f?style=for-the-badge)

**Envio um arquivo de texto ao S3 e recebo a contagem de palavras por e-mail, sem gerenciar nenhum servidor** 📄⚡📧

</div>

---

## 📌 Sobre o projeto

Neste desafio do **AWS re/Start**, criei uma função **AWS Lambda em Python** que **conta as palavras** de um arquivo de texto e informa o resultado por **e-mail**.

O fluxo é totalmente **orientado a eventos**:

1. 📤 Faço o **upload** de um arquivo `.txt` em um bucket do **Amazon S3**
2. ⚡ O S3 **aciona automaticamente** a função Lambda
3. 🧮 A função lê o arquivo, **conta as palavras** e monta a mensagem
4. 📣 A mensagem é publicada em um tópico do **Amazon SNS**
5. 📧 O SNS entrega o resultado por **e-mail**, com o assunto **Word Count Result**

A mensagem segue o formato exigido pelo desafio:

```text
The word count in the <nome-do-arquivo> file is nnn.
```

---

## 🎯 Objetivos

- ✅ Criar uma **função Lambda em Python** pelo Console da AWS
- ✅ Usar o perfil IAM existente **`LambdaAccessRole`**
- ✅ Configurar um **bucket do S3** que invoca a função quando um arquivo é enviado
- ✅ Criar um **tópico do SNS** com assinatura de **e-mail**
- ✅ Formatar a mensagem e o assunto do e-mail conforme o desafio
- ✅ Testar com arquivos de **diferentes contagens de palavras**
- ✅ Acompanhar a execução pelos **logs do CloudWatch**

---

## 🗺️ Arquitetura

```mermaid
flowchart LR
    U["👤 Usuário<br/>upload de .txt"] --> S3["🪣 Amazon S3<br/>bucket"]
    S3 -->|"evento ObjectCreated<br/>sufixo .txt"| L["⚡ AWS Lambda<br/>WordCountFunction<br/>Python 3.12"]
    L -->|"get_object"| S3
    L -->|"publish"| SNS["📣 Amazon SNS<br/>WordCountTopic"]
    SNS --> E["📧 E-mail<br/>Word Count Result"]
    L -.->|"logs"| CW["📊 CloudWatch Logs"]
    IAM["🔐 IAM<br/>LambdaAccessRole"] -.->|"permissões"| L

    style U fill:#6c757d,color:#fff,stroke:#495057
    style S3 fill:#569A31,color:#fff,stroke:#3f7523
    style L fill:#FF9900,color:#000,stroke:#cc7a00
    style SNS fill:#D6336C,color:#fff,stroke:#a8264f
    style E fill:#0d6efd,color:#fff,stroke:#0a58ca
    style CW fill:#E7157B,color:#fff,stroke:#b01060
    style IAM fill:#DD344C,color:#fff,stroke:#a8283a
```

---

## 🧰 Serviços utilizados

| 🎨 Categoria | 🔧 Serviço | 📝 Função no projeto |
|:---:|:---:|---|
| ⚡ Compute | **AWS Lambda** | Executa o código que conta as palavras |
| 🪣 Storage | **Amazon S3** | Recebe os arquivos de texto e dispara o evento |
| 📣 Mensageria | **Amazon SNS** | Entrega o resultado por e-mail |
| 🔐 Segurança | **AWS IAM** | Perfil `LambdaAccessRole` com as permissões da função |
| 📊 Monitoramento | **Amazon CloudWatch Logs** | Registra cada execução da função |
| 🐍 Linguagem | **Python 3.12 + boto3** | Código da função e acesso a S3 e SNS |

---

## ⚙️ Configuração utilizada

| 🔹 Recurso | 🔸 Valor |
|---|---|
| **Região** | `us-west-2` (Oregon): bucket, tópico e função na mesma região |
| **Função Lambda** | `WordCountFunction` · Python 3.12 · timeout de 10 s |
| **Perfil de execução** | `LambdaAccessRole` (perfil existente do laboratório) |
| **Variável de ambiente** | `TOPIC_ARN`: ARN do tópico SNS |
| **Gatilho** | S3 · evento `s3:ObjectCreated:*` · sufixo `.txt` |
| **Tópico SNS** | `WordCountTopic` · tipo **Standard** |
| **Assinatura** | Protocolo **Email** (confirmada) |
| **Assunto do e-mail** | `Word Count Result` |

---

## 💻 Código da função

```python
import os
import urllib.parse
import boto3

s3 = boto3.client("s3")
sns = boto3.client("sns")

TOPIC_ARN = os.environ["TOPIC_ARN"]


def lambda_handler(event, context):
    results = []

    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])

        # Lê o arquivo de texto do S3
        obj = s3.get_object(Bucket=bucket, Key=key)
        text = obj["Body"].read().decode("utf-8", errors="ignore")

        # Conta as palavras
        word_count = len(text.split())
        file_name = key.split("/")[-1]

        message = f"The word count in the {file_name} file is {word_count}."

        # Publica no tópico SNS
        sns.publish(
            TopicArn=TOPIC_ARN,
            Subject="Word Count Result",
            Message=message,
        )

        print(message)
        results.append(message)

    return {"statusCode": 200, "body": results}
```

**Decisões de implementação:**

| 🧩 Detalhe | 💡 Motivo |
|---|---|
| `TOPIC_ARN` em **variável de ambiente** | Evita deixar o ARN fixo no código e facilita trocar o tópico |
| `unquote_plus` na chave do objeto | O S3 envia nomes com espaços e caracteres especiais codificados |
| `for record in event["Records"]` | Trata todos os arquivos que chegarem em um mesmo evento |
| `errors="ignore"` ao decodificar | Evita que um caractere inválido interrompa a função |
| `print(message)` | Grava o resultado nos **logs do CloudWatch** |

---

## 🧪 Testes realizados

Enviei três arquivos de texto com contagens conhecidas e conferi o resultado recebido por e-mail:

| 📄 Arquivo | 📝 Conteúdo | 🎯 Esperado | ✅ Recebido |
|---|---|:---:|:---:|
| `sample1.txt` | `Hello AWS Lambda` | 3 | **3** |
| `sample2.txt` | `The quick brown fox jumps over the lazy dog` | 9 | **9** |
| `sample3.txt` | `Serverless computing lets developers run code without provisioning or managing servers and pay only for the compute time they consume` | 20 | **20** |

---

## 🪜 Passo a passo resumido

1. 📣 Criar o tópico **SNS** `WordCountTopic` e a assinatura de **e-mail**, confirmando pelo link recebido
2. 🪣 Criar o **bucket do S3** na mesma região
3. ⚡ Criar a função **`WordCountFunction`** (Python 3.12) usando o perfil **`LambdaAccessRole`**
4. 📝 Colar o código, clicar em **Deploy** e definir a variável `TOPIC_ARN`
5. 🔗 Adicionar o **gatilho do S3** (eventos de criação de objeto, sufixo `.txt`)
6. 📤 Enviar os arquivos `sample1.txt`, `sample2.txt` e `sample3.txt`
7. 📧 Conferir os e-mails **Word Count Result** com a contagem
8. 📊 Conferir os **logs** no CloudWatch

---

## 📸 Evidências

### 1️⃣ Tópico SNS criado
O tópico `WordCountTopic`, do tipo **Standard**.

![Tópico SNS](./images/01-sns-topico.png)

### 2️⃣ Confirmação da assinatura
O e-mail **AWS Notification - Subscription Confirmation**, com o link para confirmar a assinatura. Sem esse passo, o SNS não entrega as mensagens.

![Confirmação da assinatura](./images/02-sns-assinatura-confirmada.png)

### 3️⃣ Bucket do S3
O bucket criado na região **US West (Oregon) `us-west-2`**, a mesma da função e do tópico.

![Bucket do S3](./images/03-s3-bucket.png)

### 4️⃣ Código da função
O arquivo `lambda_function.py` da `WordCountFunction`, com o botão **Deploy** indicando que o código está publicado (*Current*).

![Código da função](./images/04-lambda-codigo.png)

### 5️⃣ Perfil de execução
A função usando o perfil **`LambdaAccessRole`**, que dá acesso ao S3, ao SNS e ao CloudWatch Logs.

![Perfil de execução](./images/05-lambda-configuracao.png)

### 6️⃣ Variável de ambiente
A variável **`TOPIC_ARN`**, que guarda o ARN do tópico SNS usado pelo código.

![Variável de ambiente](./images/05-1-lambda-configuracao.png)

### 7️⃣ Função com o gatilho do S3
O **Function overview** da `WordCountFunction` com o **S3** conectado como gatilho e a mensagem de que o gatilho foi adicionado com sucesso.

![Função com gatilho](./images/06-lambda-gatilho-s3.png)

### 8️⃣ Configuração do gatilho
O gatilho do S3 com o evento **`s3:ObjectCreated:*`** e o sufixo **`.txt`**, de modo que apenas arquivos de texto acionem a função.

![Configuração do gatilho](./images/07-gatilho-configuracao.png)

### 9️⃣ Arquivos enviados ao bucket
Os três arquivos de teste (`sample1.txt`, `sample2.txt` e `sample3.txt`) no bucket.

![Arquivos no bucket](./images/08-s3-arquivos.png)

### 🔟 E-mails com a contagem de palavras
Os três e-mails com o assunto **Word Count Result**, com as mensagens `...sample1.txt file is 3.`, `...sample3.txt file is 20.` e `...sample2.txt file is 9.`

![E-mails recebidos](./images/09-email-contagem.png)

### 1️⃣1️⃣ Logs no CloudWatch
O log da execução do `sample2.txt`: eventos `START`, a mensagem **"The word count in the sample2.txt file is 9."** e `END`. O relatório mostra duração de cerca de **485 ms**, com **128 MB** de memória configurada e **99 MB** usados.

![Logs no CloudWatch](./images/10-cloudwatch-logs.png)

---

## 🧠 Aprendizados

- ⚡ Como funciona uma arquitetura **orientada a eventos**: o upload no S3 dispara o código, sem servidor ligado esperando
- 🔗 Como configurar um **gatilho do S3** na Lambda, com filtro por **sufixo**
- 🔐 O papel do **perfil IAM** da função e por que o laboratório fornece o `LambdaAccessRole`
- 🧩 O uso de **variáveis de ambiente** para não deixar configurações fixas no código
- 📣 O **SNS** só entrega e-mail depois que a assinatura é **confirmada**
- 🌎 Bucket, tópico e função precisam estar na **mesma região**
- 📊 Os **logs do CloudWatch** mostram cada execução, a duração e a memória usada
- 💰 Na Lambda, o custo é por **execução e tempo**, e não por servidor ligado

---

## 🔐 Observação sobre os prints

> ⚠️ Os recursos que aparecem nas imagens pertencem a um **ambiente temporário de laboratório**, já encerrado. IDs de conta e ARNs foram cobertos. Em projetos reais, evite publicar IDs de conta, endereços de e-mail e chaves de acesso.

---

<div align="center">

### 👩‍💻 Autora

**LUCIANA RB DE ASSIS**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/lucianarbassis)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/alra-code)

⭐ Se este projeto foi útil, deixe uma estrela no repositório!

![footer](https://capsule-render.vercel.app/api?type=waving&color=232F3E&height=100&section=footer)

</div>
