📌 CDN / Cache Misconfiguration & Sensitive Headers Scanner

Este projeto realiza a validação de headers de segurança (CSP, HSTS, X-Frame-Options, entre outros), identifica caching inseguro, verifica cookies armazenados em cache e gera recomendações automáticas.

Também suporta execução via Docker, com envio opcional de e-mail dos resultados.

🧭 Funcionalidades

✔️ Validação de headers de segurança (CSP, HSTS, XFO, XSS-Protection etc.)

✔️ Identificação de misconfiguração em CDN/Cache

✔️ Verificação de cookies que foram indevidamente armazenados em cache

✔️ Geração de relatório no terminal

✔️ Envio opcional por e-mail (SMTP)

✔️ Container Docker pronto para uso

✔️ Publicado no Docker Hub:
    **`lessaayumi/cdn-cache-scanner:1.0`**

------------------------------------------------------------------------

## 📂 **Estrutura do Projeto**

    meu-scanner-cdn/
    │── scanner.py
    │── Dockerfile
    │── requirements.txt
    │── README.md

------------------------------------------------------------------------

## 🐳 **Criação do Dockerfile**

``` bash
nano Dockerfile
```

### Script Dockerfile

``` bash
# Dockerfile (Alpine 3.22)
FROM alpine:3.22

LABEL maintainer="Seu Nome <SEUEMAIL@gmail.com>"
ENV LANG=C.UTF-8 \
    PATH=/usr/local/bin:$PATH

# Instala ferramentas necessárias
RUN apk add --no-cache \
        bash \
        curl \
        perl \
        openssl \
        ca-certificates \
        tzdata \
        curl \
        bind-tools

# Diretório de trabalho e permissões
WORKDIR /app
RUN mkdir -p /app/scripts /results && chmod -R 755 /app /results

# Copia scripts para a imagem
COPY scanner.sh /app/scanner.sh
COPY send_report.sh /app/send_report.sh
COPY entrypoint.sh /app/entrypoint.sh

RUN chmod +x /app/*.sh

# Baixa sendEmail (script Perl) e torna executável
# Observação: este fetch pega a versão raw do repositório sugerido
RUN curl -sSL https://raw.githubusercontent.com/mogaal/sendemail/master/sendEmail -o /usr/local/bin/sendEmail \
    && chmod +x /usr/local/bin/sendEmail

# Ponto de entrada
ENTRYPOINT ["/app/entrypoint.sh"]

```

------------------------------------------------------------------------

## 📧 **Criação do Scanner**

``` bash
nano scanner.sh
```

``` bash
#!/usr/bin/env bash
# scanner.sh
# Recebe URL(s) por argumento(s) ou usa variável ENV TARGET_URL
# Gera relatório em /results/findings.txt e imprime no stdout

OUTFILE="./results/findings.txt"
: > "${OUTFILE}"

timestamp() { date -u +"%Y-%m-%dT%H:%M:%SZ"; }

echo "Scanner CDN/Cache Misconfigurations & Sensitive Headers" | tee -a "${OUTFILE}"
echo "Timestamp: $(timestamp)" | tee -a "${OUTFILE}"
echo "----------------------------------------" | tee -a "${OUTFILE}"

# Collect targets
TARGETS=()
if [ -n "${TARGET_URL}" ]; then
  IFS=',' read -ra arr <<< "${TARGET_URL}"
  for i in "${arr[@]}"; do TARGETS+=("$(echo $i | xargs)"); done
fi
# also add CLI args
for a in "$@"; do TARGETS+=("$a"); done

if [ ${#TARGETS[@]} -eq 0 ]; then
  echo "ERRO: Nenhuma target informada. Use TARGET_URL env (ex: https://example.com) ou passe como argumento." | tee -a "${OUTFILE}"
  exit 1
fi

check_headers() {
  url="$1"
  echo "" | tee -a "${OUTFILE}"
  echo "Target: ${url}" | tee -a "${OUTFILE}"
  echo "----------------------------------------" | tee -a "${OUTFILE}"

  # Get headers (follow redirects but only show headers)
  headers=$(curl -sSL -I -L --max-redirs 5 --write-out "\n" "${url}")
  if [ -z "$headers" ]; then
    echo "Falha ao recuperar headers para ${url}" | tee -a "${OUTFILE}"
    return
  fi

  echo "Raw headers:" | tee -a "${OUTFILE}"
  echo "$headers" | tee -a "${OUTFILE}"

  # Check CSP
  echo "" | tee -a "${OUTFILE}"
  echo "Checks:" | tee -a "${OUTFILE}"
  echo "Checks:" | tee -a "${OUTFILE}"
  echo -n "CSP: " | tee -a "${OUTFILE}"
  echo "$headers" | grep -i -m1 "^content-security-policy:" >/dev/null && echo "PRESENTE" | tee -a "${OUTFILE}" || echo "MISSING" | tee -a "${OUTFILE}"
  echo -n "HSTS (Strict-Transport-Security): " | tee -a "${OUTFILE}"
  echo "$headers" | grep -i -m1 "^strict-transport-security:" >/dev/null && echo "PRESENTE" | tee -a "${OUTFILE}" || echo "MISSING" | tee -a "${OUTFILE}"
  echo -n "X-Frame-Options / frame-ancestors: " | tee -a "${OUTFILE}"
  echo "$headers" | (grep -i -m1 "^x-frame-options:" >/dev/null || grep -i -m1 "^content-security-policy:.*frame-ancestors" >/dev/null) \
    && echo "PRESENTE" | tee -a "${OUTFILE}" || echo "MISSING" | tee -a "${OUTFILE}"

  # Check for sensitive headers (Server, X-Powered-By)
  echo -n "Exposed Server header: " | tee -a "${OUTFILE}"
  echo "$headers" | grep -i -m1 "^server:" >/dev/null && echo "SIM" | tee -a "${OUTFILE}" || echo "NAO" | tee -a "${OUTFILE}"
  echo -n "Exposed X-Powered-By header: " | tee -a "${OUTFILE}"
  echo "$headers" | grep -i -m1 "^x-powered-by:" >/dev/null && echo "SIM" | tee -a "${OUTFILE}" || echo "NAO" | tee -a "${OUTFILE}"

  # Check caching + cookies: if response has Set-Cookie and also Cache-Control public or Expires in future -> potencial inseguro
  has_cookie=$(echo "$headers" | grep -i -m1 "^set-cookie:" >/dev/null && echo yes || echo no)
  cache_public=$(echo "$headers" | grep -i -m1 "^cache-control:.*public" >/dev/null && echo yes || echo no)
  has_expires=$(echo "$headers" | grep -i -m1 "^expires:" >/dev/null && echo yes || echo no)
  if [ "$has_cookie" = "yes" ] && ([ "$cache_public" = "yes" ] || [ "$has_expires" = "yes" ]) ; then
    echo "POSSÍVEL RISCO: Cookies sendo expostos em respostas cacheáveis." | tee -a "${OUTFILE}"
    echo "Recomendação: Configurar Cache-Control: private ou no-store para respostas com Set-Cookie; evitar resposta cacheável com cookies." | tee -a "${OUTFILE}"
  else
    echo "Cache vs Cookie: OK / sem evidência imediata de cookies em respostas cacheáveis." | tee -a "${OUTFILE}"
  fi

  # Recommendations summary
  echo "" | tee -a "${OUTFILE}"
  echo "Recomendações (resumidas):" | tee -a "${OUTFILE}"
  # CSP
  if ! echo "$headers" | grep -i -m1 "^content-security-policy:" >/dev/null ; then
    echo "- Implementar Content-Security-Policy apropriada para mitigar XSS." | tee -a "${OUTFILE}"
  else
    echo "- CSP presente: revisar diretivas 'script-src', 'object-src' e 'frame-ancestors'." | tee -a "${OUTFILE}"
  fi
  # HSTS
  if ! echo "$headers" | grep -i -m1 "^strict-transport-security:" >/dev/null ; then
    echo "- Habilitar HSTS com 'max-age' adequado (ex: 31536000) e incluirSubDomains quando aplicável." | tee -a "${OUTFILE}"
  else
    echo "- HSTS presente: verificar diretiva includeSubDomains se apropriado." | tee -a "${OUTFILE}"
  fi
  # X-Frame-Options
  if ! (echo "$headers" | grep -i -m1 "^x-frame-options:" >/dev/null || echo "$headers" | grep -i -m1 "^content-security-policy:.*frame-ancestors" >/dev/null) ; then
    echo "- Bloquear clickjacking: adicionar X-Frame-Options: DENY ou usar frame-ancestors via CSP." | tee -a "${OUTFILE}"
  else
    echo "- Proteção contra frame/iframe detectada." | tee -a "${OUTFILE}"
  fi
  echo "----------------------------------------" | tee -a "${OUTFILE}"
}

for t in "${TARGETS[@]}"; do
  check_headers "$t"
done

echo ""
echo "Relatório gravado em: ${OUTFILE}"
echo ""
```

------------------------------------------------------------------------

## 🌐 **Criação do SEND REPORT**

``` bash
nano send_report.sh
```

``` bash
#!/usr/bin/env bash
# send_report.sh
# Envia findings.txt como anexo usando Gmail SMTP.

OUTFILE="/results/findings.txt"

# 1) Verifica se o arquivo existe
if [ ! -f "${OUTFILE}" ]; then
  echo "Arquivo de relatório não encontrado: ${OUTFILE}"
  exit 1
fi

# 2) Verifica destinatário
if [ -z "${RECIPIENT}" ]; then
  echo "Erro: variável RECIPIENT não definida."
  echo "Use: docker run ... -e RECIPIENT=seuemail@provedor.com"
  exit 1
fi

SMTP_HOST="${SMTP_HOST:-smtp.gmail.com}"
SMTP_PORT="${SMTP_PORT:-587}"
SMTP_USER="${SMTP_USER:-SEUEMAIL@gmail.com}"
SMTP_PASS="${SMTP_PASS:-ulmt azdw idsl yznb}"
SMTP_FROM="${SMTP_FROM:-HeaderScan <SEUEMAIL@gmail.com>}"

SMTP_CONN="${SMTP_HOST}:${SMTP_PORT}"

echo "Enviando relatório..."
echo "  Para: ${RECIPIENT}"
echo "  Via SMTP: ${SMTP_CONN}"

# Comando de envio
/usr/local/bin/sendEmail \
    -f "${SMTP_FROM}" \
    -t "${RECIPIENT}" \
    -u "Relatório - HeaderScan - $(date '+%Y-%m-%d')" \
    -m "Segue o relatório gerado pelo scanner HeaderScan." \
    -a "${OUTFILE}" \
    -s "${SMTP_CONN}" \
    -xu "${SMTP_USER}" \
    -xp "${SMTP_PASS}"

if [ $? -ne 0 ]; then
    echo "Falha ao enviar e-mail!"
    exit 2
fi
echo "E-mail enviado com sucesso!"
```

------------------------------------------------------------------------

## 📤 **Criação do entrypoint.sh**

``` bash
nano entrypoint.sh
```

``` bash
#!/usr/bin/env bash
set -e

# entrypoint.sh
# Executa scanner e, se informado, envia o relatório por e-mail.

# Parâmetros: aceita alvos como args (ou usar TARGET_URL env separado por vírgulas)
echo "Entrypoint: iniciando scanner..."
/app/scanner.sh "$@"
echo "Scanner finalizado."

# Se variável RECIPIENT estiver definida, tenta enviar
if [ -n "${RECIPIENT}" ] && [ -n "${SMTP_HOST}" ]; then
  echo "RECIPIENT definido; tentando enviar relatório por e-mail..."
  /app/send_report.sh
else
  echo "RECIPIENT ou SMTP_HOST não definidos; pulando envio de e-mail. Para enviar, defina RECIPIENT, SMTP_HOST, SMTP_PORT, SMTP_USER e SMTP_PASS."
fi

# Mantém container vivo brevemente para inspeção (opcional). Exit 0.
echo "Concluído."

```
------------------------------------------------------------------------

## 📦 **Publicação do DockerHub**

``` bash
docker login
```

Aqui irá abrir um aba no browser e será necessário o login no DockerHub para a publicação do projeto.

``` bash
docker build -t seuusuario/cdn-cache-scanner:1.0 .
```

``` bash
 docker run --rm -e SENDGRID_API_KEY="$SENDGRID_API_KEY"            -e SENDER_EMAIL="scanner@seu-dominio.com"            lessaayumi/cdn-cache-scanner:1.0 https://example.com email@exemplo.com
```

------------------------------------------------------------------------

## 🛠️ **Comandos úteis utilizados**

``` bash
docker build -t cdn-cache-scanner:1.0 .
docker run --rm cdn-cache-scanner:1.0
docker login
docker info | grep Username
docker tag cdn-cache-scanner:1.0 lessaayumi/cdn-cache-scanner:1.0
docker push SEUUSER/cdn-cache-scanner:1.0
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
Segurança da Informação
\
Docker Hub: *lessaayumi*
