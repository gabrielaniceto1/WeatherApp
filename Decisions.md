# Guia do meu raciocínio para resolver os quatro problemas

## Minha estratégia

Minha estratégia foi acompanhar o fluxo completo dos dados antes de modificar o código:

```text
OpenWeather API
→ Retrofit
→ Gson
→ modelos Java
→ Repository
→ ViewModel
→ LiveData
→ Activity
→ Adapter
→ RecyclerView
```

Para cada problema, procurei descobrir em qual etapa esse fluxo estava sendo interrompido ou utilizado incorretamente.

## 1. Buscar e exibir a pressão atmosférica

Primeiro verifiquei se a API realmente devolvia a pressão:

```json
{
  "main": {
    "pressure": 1013
  }
}
```

Depois acompanhei o valor dentro do projeto.

O modelo `WeatherInfo` já possuía:

```java
private float pressure;

public float getPressure() {
    return pressure;
}
```

O Gson já conseguia mapear automaticamente:

```text
JSON main.pressure
→ Weather.main
→ WeatherInfo.pressure
```

O layout também já possuía o `TextView`:

```java
R.id.pressure
```

Portanto, o problema não estava na API, no Retrofit, no modelo ou no layout. Estava no adapter, que exibia um valor fixo:

```java
String pressureValue = "Pressão: " + 1008.2 + " hPa";
```

Concluí que não precisava criar outro endpoint nem modificar o modelo. Bastava substituir o placeholder pelo valor recebido:

```java
int pressureHpa = Math.round(weather.getMain().getPressure());
String pressureValue = "Pressão: " + pressureHpa + " hPa";
pressure.setText(pressureValue);
```

## 2. Atualizar os dados ao clicar no Refresh

O botão já existia, mas mostrava apenas um Toast:

```java
fetchButton.setOnClickListener(v ->
        Toast.makeText(...).show()
);
```

Inicialmente considerei chamar no adapter:

```java
notifyDataSetChanged();
```

Depois percebi que esse método apenas redesenha os dados já armazenados:

```text
notifyDataSetChanged()
→ redesenha a lista atual
→ não chama a API
→ não cria novos objetos Weather
```

Portanto, o adapter não poderia ser responsável pelo refresh.

Seguindo MVVM, defini o fluxo correto:

```text
Clique na Activity
→ ViewModel inicia a consulta
→ Repository chama a API
→ LiveData publica a nova lista
→ Observer recebe a lista
→ Adapter atualiza os cartões
```

No ViewModel, criei uma ação pública:

```java
public void refreshWeather() {
    fetchAllForecasts();
}
```

Na Activity, conectei o botão:

```java
fetchButton.setOnClickListener(v ->
        mainViewModel.refreshWeather()
);
```

A atualização visual continua acontecendo por meio do observer:

```java
mainViewModel.getWeatherList().observe(this,
        weatherList -> adapter.updateWeathers(weatherList));
```

Quando a API responde, o ViewModel executa:

```java
_weatherList.setValue(updatedList);
```

O LiveData chama automaticamente o observer, que chama:

```java
adapter.updateWeathers(weatherList);
```

Por fim, o adapter executa:

```java
notifyDataSetChanged();
```

## 3. Corrigir os ícones meteorológicos

Comecei verificando como o código da API se transforma em um recurso Android.

A API devolve:

```json
{
  "weather": [
    {
      "icon": "03d"
    }
  ]
}
```

O adapter obtém:

```java
weather.getWeather().get(0).getIcon()
```

O `Utils` monta o nome:

```java
String nameDrawable = "weather_icon_" + name;
```

Portanto:

```text
03d
→ weather_icon_03d
→ R.drawable.weather_icon_03d
```

Ao comparar o nome esperado com os arquivos, encontrei:

```text
weather_icon_03dd.png
```

O arquivo possuía um `d` adicional. Por isso, quando o Android procurava `weather_icon_03d`, o recurso não existia.

Renomeei:

```text
weather_icon_03dd.png
→ weather_icon_03d.png
```

Depois conferi todos os códigos principais, li a documentação da API e percebi que faltavam:

```text
weather_icon_13d.png
weather_icon_13n.png
```

Além disso, percebi também que, de acordo com a documentação da API, as colors estavam sendo usadas em ordens diferentes. Adicionei  e corrigi os arquivos das cores para:

- `01` — céu limpo
- `02` — poucas nuvens
- `03` — nuvens dispersas
- `04` — nublado
- `09` — chuva forte
- `10` — chuva
- `11` — tempestade
- `13` — neve
- `50` — névoa

Durante essa alteração também identifiquei dois comportamentos importantes do `switch`:

- Cases duplicados não compilam.
- Sem `break`, a execução continua nos cases seguintes.

Por isso, cada condição passou a terminar com:

```java
break;
```

Se `getIdentifier()` devolver `0`, significa que o Android não encontrou o recurso.

Como melhoria futura, o `Utils` ainda pode validar explicitamente `idDrawable == 0` e usar um ícone meteorológico como fallback.

## 4. Eliminar dados duplicados

Primeiro investiguei se o adapter estava acumulando resultados.

O método atual limpa a lista antes de inserir os novos dados:

```java
weathers.clear();
weathers.addAll(weathersValues);
notifyDataSetChanged();
```

Isso mostrou que o adapter não era a origem principal das cidades repetidas.

Depois examinei as localizações no Repository:

```java
localizations.put("1", "-8.05428,-34.8813");
localizations.put("2", "-9.39416,-40.5096");
localizations.put("3", "-8.284547,-35.969863");
localizations.put("4", "-8.284547,-35.969863");
localizations.put("5", "-9.39416,-40.5096");
```

Existiam cinco chaves, mas apenas três coordenadas únicas:

```text
3 e 4 → mesmas coordenadas
2 e 5 → mesmas coordenadas
```

Percebi que o `HashMap` impede chaves repetidas, mas aceita valores repetidos. Como as chaves eram diferentes, todas as entradas eram armazenadas.

Removi as duas entradas duplicadas:

```java
localizations.put("1", "-8.05428,-34.8813");
localizations.put("2", "-9.39416,-40.5096");
localizations.put("3", "-8.284547,-35.969863");
```

O resultado passou a ser:

```text
3 entradas
3 coordenadas únicas
0 cidades duplicadas
```

Durante a investigação encontrei uma segunda forma de duplicidade: o ViewModel agenda atualizações em vários pontos:

```text
startFetching()
onSuccess()
onFailure()
refreshWeather()
```

Isso pode criar múltiplos ciclos de consultas simultâneos. Portanto, a duplicidade das cidades foi resolvida, mas o controle das requisições ainda pode ser melhorado.

As próximas melhorias são:

- Manter apenas um lugar responsável pelo agendamento.
- Remover callbacks pendentes antes de agendar outro.
- Usar `isFetching` para impedir buscas simultâneas.
- Contabilizar sucessos e falhas antes de finalizar um ciclo.
- Agendar a próxima busca uma única vez.

Dados duplicados podem ter origens diferentes:

```text
Duplicidade na configuração
→ mesmas coordenadas cadastradas mais de uma vez

Duplicidade na execução
→ mesma consulta iniciada várias vezes
```

Não basta remover itens repetidos no adapter; é necessário corrigir a origem e o controle das requisições.

## Resumo do método utilizado

Para resolver os problemas, segui este processo:

1. Reproduzir ou identificar o comportamento.
2. Localizar o dado na resposta JSON.
3. Acompanhar esse dado pelas camadas do aplicativo.
4. Encontrar o primeiro ponto em que o valor fica incorreto.
5. Aplicar a menor alteração necessária.
6. Verificar efeitos colaterais.
7. Separar correções obrigatórias de melhorias arquiteturais.

## Situação atual

| Problema | Situação |
|---|---|
| Pressão da API | Resolvido |
| Refresh pelo botão | Resolvido no fluxo básico |
| Ícones meteorológicos | Resolvido para os códigos conhecidos |
| Cidades duplicadas | Resolvido |
| Requisições/agendamentos duplicados | Ainda precisa de melhoria |

A principal evolução do raciocínio foi entender que a interface não deve buscar nem fabricar dados. O Repository obtém os dados, o ViewModel controla o estado, o LiveData comunica as mudanças e o adapter apenas apresenta o resultado.
