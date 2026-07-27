# Kronos

App de página única (`index.html`) para cadastrar medicamentos/suplementos, registrar
ciclos de aplicação e manter um histórico com controle de estoque. Os dados ficam no
Firestore, do seu próprio projeto Firebase.

## 1. Criar o projeto Firebase

1. Acesse https://console.firebase.google.com e crie um projeto novo.
2. No painel do projeto, clique em **Criar app** → ícone **Web** (`</>`).
3. Dê um nome ao app e finalize — **não** precisa ativar o Firebase Hosting.
4. Copie o objeto `firebaseConfig` mostrado na tela.

## 2. Ativar o Authentication (login)

1. No menu lateral, vá em **Build → Authentication → Get started**.
2. Na aba **Sign-in method**, ative o provedor **E-mail/senha**.
3. Não é preciso criar nenhum usuário por aqui — a própria tela de login do app
   tem a opção "Ainda não tenho conta — criar acesso" para o primeiro cadastro.
   Depois disso, se quiser, você pode voltar a esse painel e desativar a opção
   de auto-cadastro removendo esse botão do `index.html` (ou simplesmente não
   compartilhando o link com mais ninguém).

## 3. Configurar o Firestore

1. No menu lateral, vá em **Build → Firestore Database → Criar banco de dados**.
2. Escolha a região mais próxima e inicie em **modo produção**.
3. Em **Regras**, cole o conteúdo abaixo — cada usuário só consegue ler e escrever
   os próprios dados (guardados em `/users/{seu-uid}/...`), mesmo estando todos no
   mesmo projeto Firebase:

   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId}/{document=**} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```

   > Se você já usava o app antes dessa mudança, os dados antigos ficaram nas
   > coleções antigas (`substances`, `cycles`, `applications`, na raiz do banco) e
   > não aparecem mais — o app agora lê e escreve em `/users/{uid}/...`. Se quiser
   > recuperar esses dados antigos, dá pra copiá-los manualmente pelo console do
   > Firebase para dentro da pasta do seu usuário.

## 4. Editar o `index.html`

Abra o arquivo e substitua o objeto `firebaseConfig` (perto do início da tag `<script>`)
pelos valores copiados no passo 1:

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

## 5. Publicar no GitHub Pages

1. Crie um repositório novo no GitHub (pode ser privado ou público).
2. Suba o `index.html` para a raiz do repositório.
3. Vá em **Settings → Pages**.
4. Em **Source**, escolha a branch `main` e a pasta `/ (root)`.
5. Salve — em alguns minutos o app estará em
   `https://SEU_USUARIO.github.io/NOME_DO_REPOSITORIO/`.

## Como o app funciona

- **Login**: a primeira tela pede e-mail e senha. Use "criar acesso" na primeira vez;
  depois disso, é só entrar normalmente. O botão "sair" fica no topo do app.
- **Início (Dashboard)**: tela inicial com a última aplicação registrada (dose, local,
  protocolo), a previsão da próxima aplicação, o ciclo marcado como atual (com a data
  de início), um resumo dos últimos 7 dias com um mini gráfico de mg aplicados por dia,
  um alerta quando algum produto está com estoque baixo ou zerado, e a visão geral do
  estoque de todas as substâncias com barra de progresso.
- **Substâncias** (modal, aberto pelo botão flutuante): o cadastro em si é só **nome,
  apelido (opcional) e dosagem** (concentração em mg/ml) — o estoque não entra mais
  aqui. Cada substância aparece com um indicador visual do estoque restante (que
  começa zerado até você lançar uma compra).
- **Lançar estoque** (mesmo modal de Substâncias): registra uma compra — escolhe a
  substância, data da compra, quantidade (unidades), ml comprados, valor pago,
  marca e fornecedor. O mg total é calculado automaticamente pela dosagem da
  substância. Cada lançamento soma ao estoque disponível; apagar um lançamento
  desconta de volta. Fica tudo listado em "Lançamentos de compra", dentro do mesmo
  modal — dá pra ter vários lançamentos por substância, cada um com sua marca,
  fornecedor e valor.
- **Ciclos** (modal): define um nome, horário, data de início, e adiciona uma ou mais
  substâncias ao ciclo — cada substância tem sua própria dose (ml) e seu próprio
  agendamento, que pode ser **dias fixos da semana** ou **a cada N dias** (contando a
  partir do início do ciclo — por exemplo, a cada 10 dias). Dá pra misturar os dois
  tipos no mesmo ciclo. Um dos ciclos pode ser marcado como "atual" (botão "Marcar
  como atual" em cada card) — é esse que aparece no dashboard. Informando também a
  **duração em semanas**, o app calcula sozinho a data de término, quantas aplicações
  são esperadas no total, os totais previstos de ml/mg (por semana e do ciclo
  inteiro) e o **custo previsto do ciclo** (usando o preço médio por ml de cada
  substância, calculado a partir dos lançamentos de estoque que têm valor
  informado — se algum estiver sem preço lançado, o custo aparece marcado como
  "parcial") — essas previsões aparecem no card do ciclo, no relatório e na
  mensagem de WhatsApp.
- **Registrar** (modal): escolhe o ciclo e o app mostra, marcadas, as substâncias
  agendadas para o dia da data selecionada (mudar a data recalcula a lista). Dá pra
  desmarcar as que não vai aplicar e registrar várias de uma vez. Também pede o local
  de aplicação (glúteo, deltoide, vasto lateral, ventroglúteo — esquerdo/direito — ou
  outro). O app calcula o total em ml e mg antes de salvar, e desconta cada quantidade
  do estoque da respectiva substância.
- **Histórico**: lista as aplicações agrupadas por protocolo (ciclo) e, dentro de cada
  protocolo, por dia (recolhido por padrão, mostrando só a data e os totais — toque
  para expandir e ver/editar/apagar cada substância). Logo abaixo fica a lista de
  registros de sintomas.
- **Calendário de adesão** (na tela Início): visão de mês mostrando quais dias tiveram
  aplicação de fato (verde), quais estavam previstos pelo ciclo mas não foram cumpridos
  (vermelho), e quais estão previstos para o futuro (contorno tracejado).
- **Sintomas e sensações**: registro rápido (humor, energia, sono, libido de 1 a 5,
  efeitos colaterais e observações) pelo botão flutuante. Aparece listado na aba
  Histórico.
- **Perfil**: botão no cabeçalho (ícone de pessoa) para informar nome, data de
  nascimento (calcula a idade automaticamente), sexo e um **orçamento mensal (R$)**
  opcional. O sexo é usado para ajustar as faixas de referência dos exames de sangue
  (testosterona, estradiol, hemoglobina e outros marcadores que diferem entre
  homens e mulheres); o nome aparece no relatório no lugar do e-mail; o orçamento
  alimenta o alerta de gastos no Início.
- **Gastos**: no Início, um card mostra quanto foi gasto este mês, nos últimos 30
  dias e no total geral, somando os lançamentos de estoque que têm valor
  preenchido. Se você definiu um orçamento mensal no perfil, aparece uma barra de
  progresso e um aviso quando o gasto do mês ultrapassa o limite. O mesmo resumo
  (este mês / 30 dias / total) também aparece no topo de "Lançamentos de compra",
  dentro do modal de Substâncias.
- **Exames de sangue**: registro de valores de exames, pelo botão flutuante — hormonal
  (testosterona total/livre, estradiol, LH, FSH, prolactina, SHBG), perfil lipídico
  (colesterol total, HDL, LDL, triglicerídeos), função hepática (TGO, TGP, Gama GT,
  bilirrubina), função renal (creatinina, ureia, TFG), hematológico (hemoglobina,
  hematócrito, hemácias) e outros (PSA, glicemia, TSH, pressão arterial), além de data
  da coleta e laboratório. Todos os campos são opcionais — preencha só o que constar
  no seu exame. Valores fora da faixa de referência (ajustada pelo sexo informado no
  perfil, quando houver) aparecem marcados com ⚠ (essas faixas são gerais e variam por
  laboratório — não é orientação médica). O mesmo modal tem um **gráfico de
  evolução**: escolha um marcador e veja como ele mudou ao longo do tempo, com a
  faixa de referência destacada no fundo do gráfico.
- **Peso corporal**: registro rápido de peso (kg) pelo botão flutuante, com gráfico
  de evolução e variação no período, no mesmo padrão dos exames.
- **Avaliação corporal**: registro de dobras cutâneas (mm) — peitoral, tricipital,
  bicipital, axilar média, subescapular, supra-ilíaca, supra-espinhal, abdominal,
  coxa e panturrilha — e circunferências (cm) — torácico insp./exp., cintura,
  abdômen, quadril, e as medidas pareadas esquerda/direita de braço relaxado,
  braço contraído, antebraço, coxa (proximal/média/distal) e perna. Campos
  organizados em duas seções recolhíveis, todos opcionais, com data do
  lançamento. Fica listado dentro do próprio modal, com opção de apagar.
  Preenchendo as 7 dobras usadas na fórmula (peitoral, axilar média, tricipital,
  subescapular, abdominal, supra-ilíaca e coxa), o app calcula automaticamente o
  **percentual de gordura** pela equação de **Jackson & Pollock (7 dobras) + Siri**
  — precisa de sexo e data de nascimento preenchidos no Perfil pra funcionar (usados
  para calcular a densidade corporal). O valor calculado fica congelado em cada
  registro (junto com a idade e o sexo usados no cálculo daquela vez), então não
  muda se você atualizar o perfil depois.
- **Relatório**: na aba Histórico, o botão "Relatório" monta um resumo (ciclo atual,
  adesão, totais por substância, exame mais recente do período e sintomas) pronto
  para imprimir ou salvar como PDF pelo próprio navegador, ou compartilhar via
  WhatsApp — útil para levar numa consulta médica.
- **Previsão de término do estoque**: no Início e na lista de Substâncias, cada item
  mostra uma estimativa de quantos dias o estoque ainda dura, calculada pelo ritmo de
  aplicações registradas nos últimos 30 dias. Só aparece quando há histórico
  suficiente para calcular.
- **Lembretes**: o sino no topo (ao lado de "sair") ativa notificações do navegador
  perto do horário da próxima aplicação prevista. Funciona enquanto a aba/o app estiver
  aberto — é uma limitação do navegador, não dá pra disparar notificação com o app
  totalmente fechado sem um servidor por trás. Na primeira vez, o navegador vai pedir
  permissão de notificação.

### Instalar como app

O Kronos agora tem um `manifest.webmanifest` e um `sw.js` (service worker mínimo)
que permitem instalá-lo na tela inicial do celular ou do computador, abrindo em
tela cheia, sem a barra do navegador — como um app nativo. No Chrome/Edge (Android
e desktop), aparece um botão "Instalar" ou "Adicionar à tela inicial" na barra de
endereço ou no menu; no Safari/iOS, use Compartilhar → "Adicionar à Tela de Início".
Isso exige subir os arquivos `manifest.webmanifest` e `sw.js` junto com o
`index.html`, na raiz do repositório (mesma pasta, sem mudar nada de configuração
extra).

### Navegação

A navegação por abas (topo no desktop, embaixo no celular) tem só **Início** e
**Histórico**. As outras seções ficam em dois lugares diferentes, pensados pela
frequência de uso:

- **Botão flutuante** (canto inferior direito): ação direta, sem menu — toca e já
  abre "Registrar aplicação", que é o uso mais frequente do dia a dia.
- **Menu lateral** (ícone de três linhas no canto superior esquerdo do cabeçalho):
  reúne as seções de cadastro/consulta que você usa com menos frequência —
  Substâncias, Ciclos, Sintomas e sensações, Exames de sangue, Avaliação corporal
  e Peso corporal.
  Cada uma abre direto na tela com a lista e o formulário juntos.

Todos os dados sincronizam em tempo real com o Firestore — se você abrir o app em
dois dispositivos logados na mesma conta, ambos atualizam automaticamente.

Cada conta (e-mail/senha) tem seus próprios registros, isolados dos de outras
contas — se mais de uma pessoa usar o mesmo projeto Firebase, cada uma só vê e
mexe nos seus próprios cadastros, ciclos e histórico.
