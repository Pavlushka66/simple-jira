### SimpleJira

.NET Library for working with [Jira Rest Api](https://developer.atlassian.com/server/jira/platform/rest-apis/) 
which allows you to interact with the API through the LINQ interface and has full built-in support for unit testing.

### Features

- User-friendly poxy that covers the major scenarios of working with Jira API: 
    - create, read, update issues
    - search issues by JQL
    - to add a new comment to an issue
    - get a list of comments to an issue
    - upload attachments
    - download attachments
    - get a list of workflow transitions
    - invoke transition
- C# Mapping class support for easier work with issue fields
- InMemory implementation with JQL search support for objects stored in memory
- File-based implementation with JQL search support for objects stored on local drive
- Linq Provider, that support both InMemory and real implementations

### NuGet

To install [SimpleJira NuGet package](https://www.nuget.org/packages/SimpleJira/), execute the following command in the [NuGet console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)

    PM> Install-Package SimpleJira
    
or

    dotnet add package SimpleJira

for .NET CLI.

To install [SimpleJira.Fakes package](https://www.nuget.org/packages/SimpleJira.Fakes/), execute the following command in the [NuGet console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)

    PM> Install-Package SimpleJira.Fakes
    
or

    dotnet add package SimpleJira.Fakes

for .NET CLI.

## Proxy

To interact with Jira you should use IJira interface. In order to instantiate it, you need to execute the following code

```c#
IJira jira = Jira.RestApi("http://myjira.com", "my_user", "my_password");
```

#### Create new issue

```c#
JiraIssue issue = new JiraIssue {
    Project = new JiraProject {
        Key = "DEV"
    },
    IssueType = new JiraIssueType {
        Id = "22",
        Name = "Bug"
    },
    Summary = "New bug",
    Description = "New bug description",
    Assignee = new JiraUser {
        Key = "developer",
    },
    DueDate = DateTime.Today
};
issue.CustomFields[11323].Set("Custom value");

JiraIssueReference reference = await jira.CreateIssueAsync(issue, cancellationToken);
```

#### Modify issue

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");
JiraIssue issue = new JiraIssue {
    DueDate = DateTime.Today.AddDays(1)
};
await jira.UpdateIssueAsync(reference, issue, cancellationToken);
```

#### Issue search

```c#
string jql = "assignee = developer";
JiraIssuesRequest request = new JiraIssuesRequest {
    Jql = jql,
    StartAt = 0,
    MaxResults = 200,
    Fields = new string[] {"duedate", "summary", "assignee"}
};

JiraIssuesResponse response = await jira.SelectIssuesAsync(request, cancellationToken); 
```

#### Add comment for issue

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");
JiraComment comment = new JiraComment {
    Body = "New comment"
};

JiraComment newComment = await jira.AddCommentAsync(reference, comment, cancellationToken);
```

#### Read comments from issue

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");
JiraCommentsRequest request = new JiraCommentsRequest {
    StartAt = 0,
    MaxResults = 200
};

JiraCommentsResponse response = await jira.GetCommentsAsync(reference, request, cancellationToken);
```

#### Upload attachments

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");
byte[] bytes = GetContent();

JiraAttachment attachment = await jira.UploadAttachmentAsync(reference, "sample.txt", bytes, cancellationToken);
```

#### Download attachment

```c#
JiraAttachment attachment = new JiraAttachment {
    Id = "2467578"
};

byte[] bytes = await jira.DownloadAttachmentAsync(attachment, cancellationToken);
```

#### Delete attachment

```c#
JiraAttachment attachment = new JiraAttachment {
    Id = "2467578"
};

await jira.DeleteAttachmentAsync(attachment, cancellationToken);
```

#### Get a list of workflow transitions

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");

JiraTransition[] transitions = await jira.GetTransitionsAsync(reference, cancellationToken);
```

#### Invoke transition
```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");

await jira.InvokeTransitionAsync(reference, "242151256", null, cancellationToken);
```

## Mapping classes
The most convenient way to work with issues is to create mapping class for each issue type, that you have in jira.

You should declare 2 constructors for each mapping class. And you also need to declare a JiraIssuePropertyAttribute for each field. Mapping class should be inherited from the JiraIssue class 

```c#
public class JiraClientIssue : JiraIssue {
    
    public JiraClientIssue()
    {
    }

    public JiraClientIssue(IJiraIssueFieldsController controller) 
        : base(controller)
    {
    }

    // For built-in Jira field
    [JiraIssueProperty("updated")]
    public DateTime Updated
    {
        get => Controller.GetValue<DateTime>("updated");
        set => Controller.SetValue("updated", value);
    }

    // For custom-defined field
    [JiraIssueProperty(11323)]
    public string AwesomeCustomField
    {
        get => CustomFields[11323].Get<string>();
        set => CustomFields[11323].Set(value);
    }
}
```

Mapping classes make it easier, not to keep customField identifiers in mind, but to work with fields in Jira using natural metaphors.

```c#
JiraIssueReference reference = JiraIssueReference.FromKey("DEV-234761");
JiraClientIssue issue = new JiraClientIssue {
    AwesomeCustomField = "My awesome field"
};

await jira.UpdateIssueAsync(reference, issue, cancellationToken);
```

#### Scope

Usually a Mapping class is used for each issue type. However, issue type often creates restrictions on issues such as 'project' and 'issueType'. In particular, in order to create a new issue in Jira you must always specify the necessary fields. This can be annoying and it is easy to forget about this, so it recommended to declaring Scope in a Mapping class for convenience.

```c#
public class JiraClientIssue : JiraIssue {

    // put here constructors and fields

    private class Scope : IDefineScope<JiraClientIssue>
    {
        public void Build(IScopeBuilder<JiraClientIssue> builder)
        {
            builder
                .Define(x => x.Project, new JiraProject { Key = "DEV"})
                .Define(x => x.IssueType, new JiraIssueType { Id = "22", Name = "Bug" });
        }
    }
}
```   

In this case, all necessary fields will be populated when the class instance is created.

```c#
JiraClientIssue issue = new JiraClientIssue();

Console.WriteLine(issue.Project.Key);    // DEV
Console.WriteLine(issue.IssueType.Name); // Bug 
```

Plus, when you search issues via Linq-Provider, the "project = DEV AND issueType = Bug" filter will automatically be applied to the query in Jira, allowing Jira to execute the query more efficiently. 

IMPORTANT: The IJira.SelectIssuesAsync method will not apply additional filters to JQL.

## InMemory realization

При автоматическом тестировании кода, взаимодействующего с Jira, обычно нет возможности поднять тестовый instance обычной Jira. Поэтому можно использовать InMemory эмулятор Jira. И даже если этот instance будет поднят, то это может вызывать следующие проблемы для автоматического тестирования.

- Скорость. Настоящая Jira работает медленно, что неприемлемо для работы в стиле TDD, при котором крайне важно получать мнговенную обратную связь от тестов;
- Создание тестовой среды. Обычно для тестов необходимо создать только минимально необходимый пресет данных в Jira. В настоящей Jira чаще всего необходимо создавать полный пресет данных. Таким образом код теста становится перенасыщенным и перестаёт отражать суть тестируемого взаимодействия. В InMemory реализации нет дополнительных проверок на обязательность заполнения полей, которые неважны для теста.
- Независимость тестов. При использовании настоящей Jira в тестах необходимо обеспечить изоляцию тестов, в которой состояние Jira после каждого теста будет скидываться к начальному состоянию. В базах данных это может достигаться транзакциями, которых в Jira нет. InMemory реализацию можно создавать одну на каждый тест, что обеспечит пустую Jira и избавит от нежелательных side-эффектов и морганий.

#### Инициализация InMemory

```c#
JiraUser currentUser = new JiraUser {
    Key = "coolwage" 
};
JiraMetadataProvider metadata = new JiraMetadataProvider(new [] {typeof(JiraClientIssue)});

IJira jira = FakeJira.InMemory("http://fake.jira.int", currentUser, metadata);
```

## Файловая реализация

Для интеграционного тестирования необходимо, чтобы разные сервисы системы, развёрнутые в разных процессах имели доступ к одному и тому же экземпляру Jira. Дле этого создана файловая реализация эмулятора

#### Инициализация файловой реализации

```c#
var folderPath = Path.Combine(Path.GetTempPath(), "fileJiraImplementation");
JiraUser currentUser = new JiraUser {
    Key = "coolwage" 
};
JiraMetadataProvider metadata = new JiraMetadataProvider(new [] {typeof(JiraClientIssue)});

IJira jira = FakeJira.File(folderPath, "http://fake.jira.int", currentUser, metadata);
```


## Linq-Provider

Linq-Provider естественный инструмент в .NET среде, позволяющий работать с базами данных и любыми другими провайдерами данных через интерфейс Linq при наличии Object-Relational Mapping. Роль последнего в данном случае выполняет множество Mapping классов.

Для инициализации Linq-Provider необходимо создать класс JiraMetadataProvider. Который в конструкторе принимает коллекцию из всех типов Mapping классов, объявленных в проекте.

Инициализация Linq-Provider

```c#
JiraMetadataProvider metadata = new JiraMetadataProvider(new [] {typeof(JiraClientIssue)});
IJira jira = Jira.RestApi("http://myjira.com", "my_user", "my_password");

JiraQueryProvider provider = new JiraQueryProvider(jira, metadata);
``` 

Инициализация InMemory Linq-Provider

```c#
JiraUser currentUser = new JiraUser {
    Key = "coolwage" 
};
JiraMetadataProvider metadata = new JiraMetadataProvider(new [] {typeof(JiraClientIssue)});
IJira jira = FakeJira.InMemory("http://fake.jira.int", currentUser, metadata);

JiraQueryProvider provider = new JiraQueryProvider(jira, metadata);
```

###Сценарии

#### Where

```c#
JiraClientIssue[] issues = provider.GetIssues<JiraClientIssue>()
    .Where(x => x.Assignee == "coolwage")
    .ToArray();
```

Поддерживаемые операции

- x => x.Assignee == "coolwage", транслируется в "assignee = coolwage"
- x => x.DueDate = DateTime.Now, транслируется в "duedate = '2020-12-08 00:00'"
- x => JqlFunctions.Contains(x.Summary, "тест"), транслируется в "summary ~ тест"
- x => x.Labels.Contains("mylabel"), транслируется в "labels = mylabel"
- x => x.DueDate [>, <, >=, <=] DateTime.Now, транслируется в ""duedate [>, <, >=, <=] '2020-12-08 00:00'"" 

#### Select

Если не указать в запросе конструкцию Select, то Linq-Provider запросит у Jira информацию по всем полям, которые есть в Jira для текущего issue type. В случае когда необходимо выбрать большое количество issue — Jira может быть серьёзно нагружена и запрос к ней упадёт по таймауту. Поэтому rest api требует указывать поля, необходимые к выдаче для запроса. Для поддержки этого механизма в Linq-Provider включена поддержка конструкции Select.

Данный запрос вернёт summary всех issue, которые имеют issueType, определённый в Scope у Mapping класса JiraClientIssue.

```c#
string[] issues = provider.GetIssues<JiraClientIssue>()
    .Select(x => x.Summary)
    .ToArray();
``` 

В Jira уйдёт следующий запрос. Обратите внимание, что jql в данном случае будет не пустым, если в JiraClientIssue объявлен Scope

```json
{
  "jql": "project = DEV AND issueType = Клиент",
  "fields": ["summary"],
  "startAt": 0,
  "maxResults": 200
}
```

Поддерживаются следующие конструкции:

- единственное поле Select(x => x.Summary);
- анонимные типы Select(x => new { Text = x.Summary });
- обычные классы с инициализаций properties Select(x => new MyAwesomeClass {Text = x.Summary});

#### Contains

Аналог IN фильтра в JQL

```c#
JiraProject project = new JiraProject {
    Key = "DEV"
};
string[] issues = provider.GetIssues<JiraClientIssue>()
    .Where(x => new[] {project}.Contains(x.Project))
    .ToArray();
``` 

Будет транслирован в 

```json
{
  "jql": "project in (DEV)",
  "startAt": 0,
  "maxResults": 200
}
```

#### Count

Получение количества issue. При этом запрос не будет материализовывать все issue, он только запросит у Jira информацию о количестве issue

```c#
int count = provider.GetIssues<JiraClientIssue>()
    .Count();
``` 


#### Any

Получение информации о налиции issue, удовлетворяющих фильтру. При этом запрос не будет материализовывать все issue, он только запросит у Jira информацию о количестве issue

```c#
bool exists = provider.GetIssues<JiraClientIssue>()
    .Any();
``` 

#### Подзапросы

В Jira реализован механизм issueFunction, который позволяет выполнить подзапрос. В данном провайдере реализовано только два типа подзапросов, соответсвующие функциям parentsOf и subtasksOf

#### ParentsOf

```c#
JiraIssues[] issues = provider.Select<JiraIssue>()
    .Where(issue => provider.Select<JiraClientIssue>()
                        .Where(parent => parent.Assignee == "coolwage")
                        .Any(parent => issue.Parent == parent))
    .ToArray();
```

Будет транслирован в 

```json
{
  "jql": "issueFunction in parentsOf(\"assignee = coolwage\")",
  "startAt": 0,
  "maxResults": 200
}
```

#### SubtasksOf
```c#
JiraClientIssue[] issues = provider.Select<JiraClientIssue>()
    .Where(issue => provider.Select<JiraIssue>()
                        .Where(child => child.Assignee == "coolwage")
                        .Any(child => child.Parent == issue))
    .ToArray();
```

Будет транслирован в 

```json
{
  "jql": "issueFunction in subtasksOf(\"assignee = coolwage\")",
  "startAt": 0,
  "maxResults": 200
}
```
