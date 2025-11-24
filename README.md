io no terminal\
-   ✔️ Envio opcional por e-mail (SMTP)\
-   ✔️ Container Docker pronto para uso\
-   ✔️ Publicado no Docker Hub:\
    **`lessaayumi/cdn-cache-scanner:1.0`**

------------------------------------------------------------------------

## 📂 **Estrutura do Projeto**

    meu-scanner-cdn/
    │── scanner.py
    │── Dockerfile
    │── requirements.txt
    │── README.md

------------------------------------------------------------------------

## 🐳 **Executando com Docker**

### 1️⃣ Build da imagem

``` bash
docker build -t cdn-cache-scanner:1.0 .
```

### 2️⃣ Execução simples

``` bash
docker run --rm cdn-cache-scanner:1.0
```

------------------------------------------------------------------------

## 📧 **Execução com envio por e-mail**

``` bash
docker run --rm \
  -e RECIPIENT="seuemail@teste.com" \
  -e SMTP_HOST="smtp.com" \
  -e SMTP_PORT=587 \
  -e SMTP_USER="usuario" \
  -e SMTP_PASS="senha" \
  cdn-cache-scanner:1.0
```

------------------------------------------------------------------------

## 🌐 **Login no Docker Hub**

``` bash
docker login
```

Autenticação gerou:

    USING WEB-BASED LOGIN
    Your one-time device confirmation code is: VVWX-FKQW
    Login Succeeded

Verificação:

``` bash
docker info | grep Username
```

------------------------------------------------------------------------

## 📤 **Publicação no Docker Hub**

``` bash
docker push lessaayumi/cdn-cache-scanner:1.0
```

------------------------------------------------------------------------

## 📦 **Usando a Imagem do Docker Hub**

``` bash
docker pull lessaayumi/cdn-cache-scanner:1.0
docker run --rm lessaayumi/cdn-cache-scanner:1.0
```

------------------------------------------------------------------------

## 🛠️ **Comandos úteis utilizados**

``` bash
docker build -t cdn-cache-scanner:1.0 .
docker run --rm cdn-cache-scanner:1.0
docker login
docker info | grep Username
docker tag cdn-cache-scanner:1.0 lessaayumi/cdn-cache-scanner:1.0
docker push lessaayumi/cdn-cache-scanner:1.0
```

------------------------------------------------------------------------

## 📝 **Melhorias Futuras**

-   Implementar análise paralela para múltiplas URLs\
-   Exportação de relatório em PDF/HTML\
-   Dashboard Web para visualização\
-   Integração com CI/CD

------------------------------------------------------------------------

## 👩🏻‍💻 **Autora**

**Alessandra Lessa** and **Taynara Castilho**\
Segurança da Informação • Pesquisadora em ML aplicado à detecção de
ataques\
Docker Hub: *lessaayumi*
