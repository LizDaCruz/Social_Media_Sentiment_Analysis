open FSharp.Data
open Newtonsoft.Json
open System
open System.Net.Http
open System.Net.Http.Headers
open System.Threading.Tasks
open Microsoft.ML
open Microsoft.ML.Data
open Microsoft.ML.Transforms.Text
open XPlot.Plotly
open Giraffe
open Saturn
open Azure.AI.TextAnalytics

type SocialMediaData = {
    Id: string
    User: string
    Content: string
    Timestamp: System.DateTime
    Sentiment: string
}

type SentimentData = {
    [<LoadColumn(0)>] Sentiment: string
    [<LoadColumn(1)>] Content: string
}

type SentimentPrediction = {
    [<ColumnName("PredictedLabel")>] Sentiment: string
}

type InstagramPost = {
    id: string
    username: string
    caption: string
    timestamp: string
}

type InstagramApiResponse = {
    data: InstagramPost[]
}

type TwitterPost = {
    id: string
    text: string
    author_id: string
    created_at: string
}

type TwitterApiResponse = {
    data: TwitterPost[]
}

let fetchInstagramDataAsync (accessToken: string) = async {
    let httpClient = new HttpClient()
    let requestUrl = sprintf "https://graph.instagram.com/me/media?fields=id,username,caption,timestamp&access_token=%s" accessToken
    let! response = httpClient.GetStringAsync(requestUrl) |> Async.AwaitTask
    let data = JsonConvert.DeserializeObject<InstagramApiResponse>(response)
    return data.data |> Array.toList |> List.map (fun post ->
        { Id = post.id; User = post.username; Content = post.caption; Timestamp = DateTime.Parse(post.timestamp); Sentiment = "" })
}

let fetchTwitterDataAsync (bearerToken: string) = async {
    let httpClient = new HttpClient()
    httpClient.DefaultRequestHeaders.Authorization <- AuthenticationHeaderValue("Bearer", bearerToken)
    let requestUrl = "https://api.twitter.com/2/tweets?ids=YOUR_TWEET_IDS&tweet.fields=created_at,text,author_id"
    let! response = httpClient.GetStringAsync(requestUrl) |> Async.AwaitTask
    let data = JsonConvert.DeserializeObject<TwitterApiResponse>(response)
    return data.data |> Array.toList |> List.map (fun post ->
        { Id = post.id; User = post.author_id; Content = post.text; Timestamp = DateTime.Parse(post.created_at); Sentiment = "" })
}

let analyzeSentimentAzure (endpoint: string) (apiKey: string) (content: string) =
    let client = TextAnalyticsClient(Uri(endpoint), AzureKeyCredential(apiKey))
    let document = TextDocumentInput("1", content)
    let response = client.AnalyzeSentiment(document)
    match response.Value.Sentiment with
    | TextSentiment.Positive -> "Positive"
    | TextSentiment.Negative -> "Negative"
    | _ -> "Neutral"

let visualizeSentimentAnalysis data =
    let sentimentCounts = 
        data
        |> Seq.groupBy (fun post -> post.Sentiment)
        |> Seq.map (fun (sentiment, posts) -> sentiment, Seq.length posts)
        |> Seq.toList

    let sentimentChart =
        Chart.Bar(sentimentCounts)
        |> Chart.WithTitle "Sentiment Analysis"

    sentimentChart.Show()

let webApp =
    choose [
        route "/" >=> text "Social Media Sentiment Analysis"
        route "/data" >=> json twitterData
    ]

let app = application {
    use_router webApp
    url "http://localhost:8080"
    memory_cache
    use_gzip
}

[<EntryPoint>]
let main argv =
    let accessToken = "YOUR_INSTAGRAM_ACCESS_TOKEN"
    let bearerToken = "YOUR_TWITTER_BEARER_TOKEN"
    let azureEndpoint = "YOUR_AZURE_ENDPOINT"
    let azureApiKey = "YOUR_AZURE_API_KEY"

    // Async data fetching
    let instagramTask = fetchInstagramDataAsync accessToken |> Async.StartAsTask
    let twitterTask = fetchTwitterDataAsync bearerToken |> Async.StartAsTask

    // Wait for all data to be fetched
    let allData = Task.WhenAll(instagramTask, twitterTask).Result |> Array.concat

    // Parallel sentiment analysis using Azure Cognitive Services
    let analyzedData = 
        allData 
        |> Array.Parallel.map (fun post -> 
            { post with Sentiment = analyzeSentimentAzure azureEndpoint azureApiKey post.Content })

    // Visualization
    visualizeSentimentAnalysis analyzedData

    // Start Web Application
    run app
    0
