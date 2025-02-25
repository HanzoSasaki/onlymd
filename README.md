# 📌 Mural de Reclamações para Condomínios

Passo a Passo de construção de um mural de reclamações em baixo nivel com 2 camadas**.

---

## 🚀 1. Configuração Inicial do Projeto

### 📂 Criando a Estrutura do Projeto
```bash
mkdir mural-reclamacoes && cd mural-reclamacoes
npm init -y
```

### 📦 Instalando Dependências
```bash
npm install express mongoose nodemailer cors dotenv body-parser socket.io
```

- `express`: Servidor web  
- `mongoose`: Banco de dados MongoDB  
- `nodemailer`: Envio de e-mails  
- `cors`: Permite comunicação entre frontend e backend  
- `dotenv`: Variáveis de ambiente  
- `body-parser`: Processa JSON nas requisições  
- `socket.io`: Atualizações em tempo real para o mural  

---

## 🖥️ 2. Criando o Servidor Backend
Crie o arquivo `server.js` e adicione o seguinte código:

```javascript
require("dotenv").config();
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const nodemailer = require("nodemailer");
const http = require("http");
const { Server } = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());

mongoose.connect("mongodb://127.0.0.1:27017/mural", {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const ReclamacaoSchema = new mongoose.Schema({
  titulo: String,
  descricao: String,
  status: { type: String, default: "Em processo" },
  dataCriacao: { type: Date, default: Date.now },
  emailSolicitante: String,
  emailResponsavel: String,
});
const Reclamacao = mongoose.model("Reclamacao", ReclamacaoSchema);

app.post("/reclamacoes", async (req, res) => {
  const reclamacao = new Reclamacao(req.body);
  await reclamacao.save();
  io.emit("novaReclamacao", reclamacao);
  enviarEmail(reclamacao);
  res.status(201).json(reclamacao);
});

app.get("/reclamacoes", async (req, res) => {
  const reclamacoes = await Reclamacao.find();
  res.json(reclamacoes);
});

app.put("/reclamacoes/:id", async (req, res) => {
  const { status } = req.body;
  const reclamacao = await Reclamacao.findByIdAndUpdate(req.params.id, { status }, { new: true });
  io.emit("atualizaReclamacao", reclamacao);
  res.json(reclamacao);
});

async function enviarEmail(reclamacao) {
  let transporter = nodemailer.createTransport({
    service: "gmail",
    auth: {
      user: process.env.EMAIL,
      pass: process.env.EMAIL_PASS,
    },
  });

  await transporter.sendMail({
    from: process.env.EMAIL,
    to: reclamacao.emailResponsavel,
    subject: "Nova Reclamação Recebida",
    text: `Uma nova reclamação foi registrada:\n\nTítulo: ${reclamacao.titulo}\nDescrição: ${reclamacao.descricao}`,
  });
}

server.listen(3000, () => console.log("Servidor rodando na porta 3000"));
```

---

## 🎨 3. Criando o Frontend (HTML, CSS, JavaScript)
Crie a pasta `public` e dentro dela, os arquivos `index.html`, `style.css` e `script.js`.

### 📝 `index.html`
```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mural de Reclamações</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h1>Mural de Reclamações</h1>
    <form id="reclamacao-form">
        <input type="text" id="titulo" placeholder="Título" required>
        <textarea id="descricao" placeholder="Descrição" required></textarea>
        <input type="email" id="emailSolicitante" placeholder="Seu E-mail" required>
        <input type="email" id="emailResponsavel" placeholder="E-mail do Responsável" required>
        <button type="submit">Enviar</button>
    </form>
    <div id="mural"></div>
    <script src="/socket.io/socket.io.js"></script>
    <script src="script.js"></script>
</body>
</html>
```

### 🎨 `style.css`
```css
body {
    font-family: Arial, sans-serif;
    text-align: center;
}
#mural {
    width: 80%;
    margin: auto;
}
.reclamacao {
    border: 1px solid #ccc;
    padding: 10px;
    margin: 10px;
}
.atrasado {
    background: red;
    color: white;
}
```

### 🛠️ `script.js`
```javascript
const socket = io();

document.getElementById("reclamacao-form").addEventListener("submit", async (e) => {
    e.preventDefault();
    const titulo = document.getElementById("titulo").value;
    const descricao = document.getElementById("descricao").value;
    const emailSolicitante = document.getElementById("emailSolicitante").value;
    const emailResponsavel = document.getElementById("emailResponsavel").value;

    const response = await fetch("http://localhost:3000/reclamacoes", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ titulo, descricao, emailSolicitante, emailResponsavel }),
    });

    if (response.ok) location.reload();
});

async function carregarReclamacoes() {
    const response = await fetch("http://localhost:3000/reclamacoes");
    const reclamacoes = await response.json();
    const mural = document.getElementById("mural");
    mural.innerHTML = "";

    reclamacoes.forEach((rec) => {
        const div = document.createElement("div");
        div.className = "reclamacao " + (rec.status === "Atrasado" ? "atrasado" : "");
        div.innerHTML = `<h3>${rec.titulo}</h3><p>${rec.descricao}</p><p>Status: ${rec.status}</p>`;
        mural.appendChild(div);
    });
}

socket.on("novaReclamacao", carregarReclamacoes);
socket.on("atualizaReclamacao", carregarReclamacoes);

carregarReclamacoes();
```

