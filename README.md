# MyUniHub

Um hub universitário completo para gerenciamento de estudos, com recursos de:

- 📚 Gerenciamento de matérias e livros
- 📅 Controle de prazos e tarefas  
- 📊 Dashboard de progresso com gráficos
- 📖 Biblioteca pessoal com anotações
- ⏱️ Técnica Pomodoro integrada
- 🔐 Autenticação segura com Firebase
- 💾 Persistência de dados em tempo real com Firestore

## 🛠️ Tecnologias

- HTML5 + CSS3 (Tailwind CSS)
- JavaScript ES Modules
- Chart.js para visualizações
- Firebase (Auth, Firestore)
- Design responsivo

## 🔒 Security

> ⚠️ **Importante sobre segurança**: Este aplicativo cliente expõe a chave de API do Firebase no código-fonte (`index.html`). Este é um padrão esperado para aplicações Firebase client-side. A segurança depende de:
>
> 1. **Restrições de chave API no Google Cloud Console**: A chave tem restrição de HTTP referrer (permitindo apenas `*.web.app`, `*.firebaseapp.com` e `localhost`) e restrição de API (permitindo apenas serviços Firebase utilizados).
> 2. **Firestore Security Rules**: Regras robustas que garantem isolamento por usuário (`uid`) e bloqueiam acessos anônimos ou não autorizados.
> 3. **Subresource Integrity (SRI)**: Todos os scripts externos carregados via CDN possuem verificação de integridade para impedir ataques à cadeia de suprimentos.
>
> Para detalhes completos sobre nossas práticas de segurança, consulte [SECURITY.md](SECURITY.md).

## 🚀 Como usar

1. Abra `index.html` em um navegador moderno
2. Para usar recursos que requerem autenticação, faça login com credenciais válidas do Firebase
3. O aplicativo funciona totalmente offline após o primeiro carregamento (com sincronização quando online)

## 📝 Licença

Este projeto é parte de um exercício acadêmico de desenvolvimento seguro.