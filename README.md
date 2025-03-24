# YouTube to MP3 API

Uma API robusta e eficiente para baixar vídeos do YouTube, extrair o áudio em formato MP3 e disponibilizar através de URLs temporárias.

![YouTube to MP3](https://img.shields.io/badge/YouTube-MP3-red)
![Node.js](https://img.shields.io/badge/Node.js-14%2B-green)
![Express](https://img.shields.io/badge/Express-4.x-blue)

## ✨ Características

- ⬇️ Download de vídeos do YouTube com múltiplas abordagens de fallback
- 🔄 Suporte a proxy residencial para contornar restrições anti-bot
- 🎵 Conversão para MP3 em alta qualidade
- 🔗 URLs temporárias para download
- ⚡ Processamento assíncrono com acompanhamento de status
- ⏱️ Expiração automática após 1 hora
- 🧹 Limpeza automática de arquivos temporários

## 📋 Pré-requisitos

- Node.js (versão 14 ou superior)
- npm ou yarn
- FFmpeg instalado no sistema
- Python 3 e pip (para yt-dlp)

## 🔧 Instalação

### 1. Clone o repositório

```bash
git clone https://github.com/seu-usuario/youtube-to-mp3-api.git
cd youtube-to-mp3-api
```

### 2. Instale as dependências

```bash
# Instalar dependências do Node.js
npm install

# Instalar yt-dlp (substituto moderno do youtube-dl)
pip3 install --upgrade yt-dlp

# Instalar FFmpeg (se ainda não tiver)
# Para Ubuntu/Debian:
sudo apt update
sudo apt install -y ffmpeg
```

### 3. Configure o ambiente

A aplicação está configurada para usar um proxy residencial que ajuda a contornar as restrições anti-bot do YouTube. O proxy padrão está configurado no código, mas você pode modificá-lo se necessário.

### 4. Inicie o servidor

```bash
# Diretamente:
node index.js

# Ou com PM2 para manter rodando em segundo plano:
pm2 start index.js --name yt2mp3
```

Por padrão, o servidor iniciará na porta 3000. Você pode alterar isso definindo a variável de ambiente `PORT`.

## 📝 Como usar

### Converter um vídeo do YouTube para MP3

**Endpoint:** `POST /convert`

**Corpo da requisição (JSON):**
```json
{
  "youtubeUrl": "https://www.youtube.com/watch?v=exemplo"
}
```

**Resposta de sucesso:**
```json
{
  "success": true,
  "message": "Tarefa de download iniciada",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "statusUrl": "/status/123e4567-e89b-12d3-a456-426614174000",
  "downloadUrl": "/download/123e4567-e89b-12d3-a456-426614174000",
  "estimatedDuration": "Alguns minutos, dependendo do tamanho do vídeo"
}
```

### Verificar o status da tarefa

**Endpoint:** `GET /status/:taskId`

**Resposta de sucesso:**
```json
{
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "status": "completed",
  "title": "Título do Vídeo",
  "created": "2023-03-21T12:34:56.789Z",
  "downloadUrl": "/download/123e4567-e89b-12d3-a456-426614174000",
  "error": null
}
```

Possíveis valores para `status`:
- `pending`: A tarefa está na fila para ser processada
- `completed`: O download e a conversão foram concluídos com sucesso
- `failed`: Ocorreu um erro durante o processamento

### Baixar o arquivo MP3

**Endpoint:** `GET /download/:fileId`

Use a URL fornecida na resposta anterior para baixar o arquivo MP3.

### Verificar status da API

**Endpoint:** `GET /status`

**Resposta:**
```json
{
  "status": "online",
  "version": "1.1.0",
  "message": "API funcionando normalmente"
}
```

## 📊 Exemplo de uso com Node.js

```javascript
const axios = require('axios');
const fs = require('fs');

// URL base da API
const API_URL = 'http://localhost:3000';

async function convertAndDownload(youtubeUrl, outputPath) {
  // Etapa 1: Solicitar a conversão
  const conversion = await axios.post(`${API_URL}/convert`, {
    youtubeUrl: youtubeUrl
  });
  
  const taskId = conversion.data.taskId;
  console.log(`Tarefa iniciada: ${taskId}`);
  
  // Etapa 2: Verificar o status periodicamente
  let completed = false;
  
  while (!completed) {
    await new Promise(resolve => setTimeout(resolve, 5000)); // Esperar 5 segundos
    
    const statusResponse = await axios.get(`${API_URL}/status/${taskId}`);
    const status = statusResponse.data;
    
    console.log(`Status: ${status.status}`);
    
    if (status.status === 'completed') {
      completed = true;
    } else if (status.status === 'failed') {
      throw new Error(`Falha no processamento: ${status.error}`);
    }
  }
  
  // Etapa 3: Baixar o arquivo
  const response = await axios({
    method: 'GET',
    url: `${API_URL}/download/${taskId}`,
    responseType: 'stream'
  });
  
  const writer = fs.createWriteStream(outputPath);
  response.data.pipe(writer);
  
  return new Promise((resolve, reject) => {
    writer.on('finish', resolve);
    writer.on('error', reject);
  });
}

// Uso:
convertAndDownload('https://www.youtube.com/watch?v=exemplo', './musica.mp3')
  .then(() => console.log('Download completo!'))
  .catch(console.error);
```

## 📚 Estrutura do projeto

```
youtube-to-mp3-api/
├── index.js          # Arquivo principal da API
├── package.json      # Dependências e scripts
└── temp/             # Diretório para arquivos temporários (criado automaticamente)
```

## 🔄 Sistema de contorno de restrições

Esta versão aprimorada da API utiliza um sistema de múltiplas abordagens para garantir o download mesmo quando o YouTube restringe acessos:

1. **Abordagem 1**: Utiliza proxy residencial para contornar as restrições anti-bot
2. **Abordagem 2**: Combina proxy residencial com alternativas do Invidious (front-ends alternativos do YouTube)
3. **Abordagem 3**: Configurações avançadas para yt-dlp que contornam restrições
4. **Abordagem 4**: Utiliza o YouTube Music como alternativa (às vezes tem menos restrições)
5. **Abordagem 5**: Tenta download através do Piped.video (outro front-end alternativo)

Esse sistema de fallback aumenta significativamente a taxa de sucesso nos downloads, mesmo com as restrições anti-bot do YouTube.

## 🔀 Processamento Assíncrono

A API implementa um sistema de processamento assíncrono para melhorar a experiência do usuário e permitir o download de vídeos maiores:

1. Quando uma solicitação é recebida, a API inicia o processamento em segundo plano
2. Uma resposta imediata é enviada com o ID da tarefa e URLs para verificar o status e baixar o arquivo quando estiver pronto
3. O cliente pode verificar periodicamente o status da tarefa usando o endpoint `/status/:taskId`
4. Quando o processamento for concluído, o arquivo estará disponível para download por até uma hora

Isso evita timeouts em conexões HTTP e proporciona uma experiência mais robusta para o usuário.

## 📝 Notas importantes

- Os arquivos são automaticamente excluídos após uma hora para economizar espaço em disco.
- Esta API é apenas para uso educacional. Respeite os direitos autorais e os termos de serviço do YouTube.
- Considere implementar autenticação e limitação de taxa (rate limiting) em ambientes de produção.

## 🔧 Solução de problemas

### Erro "Sign in to confirm you're not a bot"
Esta versão resolve esse problema usando proxy residencial. Se ainda encontrar esse erro:
- Verifique se o serviço de proxy está ativo e funcionando
- Tente outro proxy residencial se necessário

### Erro "FFmpeg não encontrado"
Certifique-se de que o FFmpeg está instalado corretamente:
```bash
ffmpeg -version
```

### Processo de conversão lento
O tempo de processamento depende do tamanho do vídeo original e da capacidade do servidor, além do roteamento através do proxy.

### Tarefa presa no status "pending"
Se a tarefa ficar presa no status "pending" por muito tempo:
- Verifique os logs do servidor para possíveis erros
- Reinicie o servidor se necessário
- Certifique-se de que o sistema tem recursos suficientes (CPU/RAM) para processar o vídeo

## 🚀 Possíveis melhorias

- [ ] Adicionar autenticação para proteger a API
- [ ] Implementar limitação de taxa (rate limiting)
- [ ] Adicionar suporte para diferentes formatos de áudio
- [ ] Criar um sistema de fila para processar múltiplas solicitações
- [ ] Implementar cache para vídeos frequentemente solicitados
- [ ] Adicionar suporte para transcrição de áudio para texto
- [ ] Criar um frontend web para interface de usuário

## 📄 Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).

## ⚠️ Aviso legal

Esta API é fornecida apenas para fins educacionais. O download de conteúdo protegido por direitos autorais sem a permissão dos detentores dos direitos pode violar leis de direitos autorais. Os usuários são responsáveis por garantir que seu uso desta API esteja em conformidade com as leis e regulamentos aplicáveis.

## 🔒 Configuração do Proxy Residencial

Esta versão da API está configurada para usar um proxy residencial da iProyal para contornar as restrições anti-bot do YouTube. Os proxies residenciais funcionam usando IPs de usuários reais, que são tratados com menos restrições pelo YouTube em comparação com IPs de datacenter.

Se você precisar atualizar as credenciais do proxy, edite as linhas no arquivo `index.js` que contêm a URL do proxy:

```javascript
const proxyUrl = 'http://seu_usuario:sua_senha@geo.iproyal.com:12321';
```

Para obter um proxy residencial:
1. Crie uma conta em um provedor como iProyal, Bright Data, Oxylabs, etc.
2. Configure um proxy residencial para streaming
3. Obtenha as credenciais e o endpoint
4. Substitua no código conforme necessário
