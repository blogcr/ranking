
# Critério Antiburla do Ranking

Para evitar manipulação do ranking por meio de múltiplos refreshes da mesma página, a métrica utilizada no Google Analytics deve ser:

activeUsers

Justificativa:

* screenPageViews contabiliza todas as visualizações, incluindo múltiplos reloads da mesma pessoa.
* activeUsers contabiliza usuários únicos no período.
* Se alguém der 30 refreshes seguidos, continuará sendo contado como 1 usuário.

Alteração necessária no Apps Script:

De:

metrics: [{ name: "screenPageViews" }]

Para:

metrics: [{ name: "activeUsers" }]

E também em orderBys:

metricName: "activeUsers"

Objetivo editorial:

Garantir justiça entre autores e evitar distorções artificiais no ranking semanal.

---

Perfeito.
Aqui está o **Apps Script completo**, já ajustado para usar **activeUsers** (usuários únicos), mantendo:

– janela móvel de 6 dias
– ordenação correta
– cruzamento com feed do Blogger
– retorno com título, autor (marcador) e imagem

Substitua integralmente o seu Código.gs por este:

---

```javascript
function doGet() {

  var propertyId = "398908106"; // confirme se é o ID correto

  var hoje = new Date();
  var seisDiasAtras = new Date();
  seisDiasAtras.setDate(hoje.getDate() - 6);

  var request = {
    dateRanges: [{
      startDate: Utilities.formatDate(seisDiasAtras, "GMT", "yyyy-MM-dd"),
      endDate: Utilities.formatDate(hoje, "GMT", "yyyy-MM-dd")
    }],
    dimensions: [{ name: "pagePath" }],
    metrics: [{ name: "activeUsers" }],
    orderBys: [{
      metric: { metricName: "activeUsers" },
      desc: true
    }],
    limit: 20
  };

  var response = AnalyticsData.Properties.runReport(
    request,
    "properties/" + propertyId
  );

  var resultado = [];

  if (!response.rows) {
    return ContentService
      .createTextOutput(JSON.stringify(resultado))
      .setMimeType(ContentService.MimeType.JSON);
  }

  var feed = UrlFetchApp.fetch("https://www.carlosromero.com.br/feeds/posts/default?alt=json&max-results=100");
  var feedJson = JSON.parse(feed.getContentText());
  var posts = feedJson.feed.entry;

  response.rows.forEach(function(row) {

    var path = row.dimensionValues[0].value;
    var usuarios = parseInt(row.metricValues[0].value, 10);

    if (path.indexOf("/20") === -1) return;

    var fullUrl = "https://www.carlosromero.com.br" + path;

    var match = posts.find(function(post) {
      var link = post.link.find(l => l.rel === "alternate").href;
      return link === fullUrl;
    });

    if (match) {

      var imagem = "";
      if (match.media$thumbnail) {
        imagem = match.media$thumbnail.url.replace("s72-c","s1200");
      }

      var autorReal = "";
      if (match.category && match.category.length > 0) {
        autorReal = match.category[0].term;
      }

      resultado.push({
        url: fullUrl,
        views: usuarios,
        title: match.title.$t,
        author: autorReal,
        image: imagem
      });
    }

  });

  resultado.sort(function(a, b) {
    return b.views - a.views;
  });

  return ContentService
    .createTextOutput(JSON.stringify(resultado.slice(0,10)))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

Depois:

1. Salvar
2. Implantar
3. Nova implantação
4. Atualizar
5. Usar a nova URL no GitHub

Agora o ranking passa a medir:

Usuários únicos no ciclo editorial de 6 dias.

Se alguém der 30 refreshes, continuará sendo 1.

Isso deixa o sistema muito mais equilibrado.

Se quiser, no próximo passo podemos discutir se vale a pena usar “engagedSessions” para medir leitura real em vez de apenas acesso.

