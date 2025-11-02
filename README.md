<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Chat Online P2P (ID Persistente)</title>
<!-- Inclui a biblioteca PeerJS -->
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
  /* Estilos para Dark Mode e Mobile-First */
  body {
    font-family: Arial, sans-serif;
    background-color: #0d1117;
    color: white;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start; /* Alinha o conteúdo mais ao topo */
    min-height: 100vh;
    padding: 20px 10px;
    box-sizing: border-box;
  }
  h2 {
    color: #58a6ff;
    margin-bottom: 15px;
  }
  #chat {
    width: 100%;
    max-width: 400px;
    height: 40vh; /* Altura responsiva */
    background: #161b22;
    border-radius: 10px;
    padding: 10px;
    overflow-y: auto;
    margin-bottom: 10px;
    box-shadow: 0 0 10px #000;
  }
  input, button {
    padding: 10px;
    border: none;
    border-radius: 5px;
    outline: none;
    font-size: 16px;
    box-sizing: border-box;
  }
  #msg {
    flex-grow: 1; /* Ocupa o espaço restante */
    background-color: #f0f6fc;
  }
  button {
    background: #238636;
    color: white;
    cursor: pointer;
    width: 80px;
    margin-left: 5px;
  }
  button:hover {
    background: #2ea043;
  }
  .msg {
    background: #21262d;
    padding: 6px 10px;
    border-radius: 5px;
    margin: 5px 0;
    word-wrap: break-word;
    font-size: 14px;
  }
  .nome {
    color: #58a6ff;
    font-weight: bold;
  }
  #meuId {
    font-size: 14px;
    background: #21262d;
    padding: 8px 10px;
    border-radius: 5px;
    margin-bottom: 15px;
    width: 100%;
    max-width: 400px;
    text-align: center;
  }
  #conectarId {
    flex-grow: 1;
    background-color: #f0f6fc;
  }
  .botoes-conexao {
    display: flex;
    margin-bottom: 10px;
    width: 100%;
    max-width: 400px;
    justify-content: space-between;
  }
  .botoes-conexao button {
    width: 100px;
    margin-left: 5px;
  }
  .campo-msg {
    display: flex;
    width: 100%;
    max-width: 400px;
    justify-content: space-between;
    padding-top: 10px;
  }
</style>
</head>
<body>
  <h2>💬 Chat P2P (ID Persistente)</h2>
  <div id="meuId">Aguardando conexão P2P...</div>

  <div class="botoes-conexao">
    <input id="conectarId" placeholder="ID do amigo" type="text">
    <button onclick="conectar()">Conectar</button>
  </div>

  <div id="chat"></div>

  <div class="campo-msg">
    <input id="msg" placeholder="Digite sua mensagem..." type="text">
    <button onclick="enviar()">Enviar</button>
  </div>

<script>
  // Variáveis e Configuração
  const ID_PERSISTENTE_KEY = "p2p_chat_id";
  let meuIDSalvo = localStorage.getItem(ID_PERSISTENTE_KEY);
  
  // Tenta iniciar com o ID salvo ou gera um novo
  const peer = meuIDSalvo ? new Peer(meuIDSalvo) : new Peer();
  
  const chat = document.getElementById("chat");
  const meuId = document.getElementById("meuId");
  const msgInput = document.getElementById("msg");
  const conectarInput = document.getElementById("conectarId");

  let conexao;
  // Pede o nome do usuário
  const nome = prompt("Digite seu nome:");

  // --- Funções de Ajuda ---

  function adicionarMensagem(remetente, texto) {
    const div = document.createElement("div");
    div.classList.add("msg");
    div.innerHTML = `<span class="nome">${remetente}:</span> ${texto}`;
    chat.appendChild(div);
    chat.scrollTop = chat.scrollHeight;
  }

  function receber(data) {
    adicionarMensagem(data.nome, data.texto);
  }

  // --- Lógica PeerJS ---

  // 1. Peer conectado e ID obtido/reutilizado
  peer.on("open", id => {
    if (id) {
        // Salva o ID no localStorage para persistir entre sessões
        localStorage.setItem(ID_PERSISTENTE_KEY, id);
        meuId.innerHTML = `Seu ID (Persistente): <b>${id}</b><br>Envie esse ID pro seu amigo!`;
        adicionarMensagem("Sistema", "Conexão P2P pronta. ID obtido.");
    }
  });

  // 2. Tratamento de Erro do PeerJS (Crucial para WebViews)
  peer.on('error', err => {
    console.error("Erro do PeerJS:", err);
    
    let errorMsg = `Erro: ${err.type}. Tente recarregar a página.`;

    if (err.type === 'unavailable-id') {
        // Se o ID salvo não está mais disponível, limpa e pede para o usuário recarregar
        localStorage.removeItem(ID_PERSISTENTE_KEY);
        errorMsg = "Seu ID salvo não está disponível. Recarregue a página para obter um novo.";
    } else if (err.type === 'browser-incompatible') {
        errorMsg = "ATENÇÃO: Este dispositivo (ou o App) não suporta WebRTC/PeerJS. O chat P2P não irá funcionar.";
    } else if (err.type === 'network') {
        errorMsg = "Problema de rede ou o servidor de sinalização PeerJS está inacessível.";
    }
    
    adicionarMensagem("Sistema", errorMsg);
  });

  // 3. Conexão recebida de um Peer remoto
  peer.on("connection", conn => {
    conexao = conn;
    conexao.on("open", () => {
        adicionarMensagem("Sistema", "Amigo conectado!");
    });
    conexao.on("data", receber);
    conexao.on("close", () => {
        adicionarMensagem("Sistema", "Amigo desconectou.");
    });
  });

  // 4. Inicia a conexão com um ID remoto
  function conectar() {
    const id = conectarInput.value.trim();
    if (id && id !== peer.id) {
      conexao = peer.connect(id);
      conexao.on("open", () => {
        adicionarMensagem("Sistema", "Conectado ao amigo!");
      });
      conexao.on("data", receber);
      conexao.on("close", () => {
        adicionarMensagem("Sistema", "Amigo desconectou.");
      });
      conexao.on("error", err => {
        adicionarMensagem("Sistema", `Erro na conexão: ${err.message}`);
      });
    } else if (id === peer.id) {
        adicionarMensagem("Sistema", "Não pode conectar ao seu próprio ID.");
    } else {
        adicionarMensagem("Sistema", "Por favor, digite um ID válido.");
    }
  }

  // 5. Envia a mensagem
  function enviar() {
    const texto = msgInput.value.trim();
    if (texto && conexao && conexao.open) {
      const data = { nome, texto };
      conexao.send(data);
      adicionarMensagem(nome, texto);
      msgInput.value = "";
    } else if (texto && !conexao) {
        adicionarMensagem("Sistema", "Você precisa se conectar a um amigo primeiro.");
    }
  }

  // Permite enviar a mensagem pressionando Enter
  msgInput.addEventListener("keydown", e => {
    if (e.key === "Enter") {
        e.preventDefault();
        enviar();
    }
  });
</script>
</body>
</html>
