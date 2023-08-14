---
layout: post
title: Using fake data and webhook.site for exploration
description: A simple way to generate testing data and sending them via HTTP.
author: Filip Dorian Pindej
---

## The struggle

Often times I find myself wanting to test some code snippet I just created. Reasons my vary: I may have forgotten something I haven't used in a long time, I want to validate something...

Creating Your own data is annoying, often time consuming and demotivating. Not to mention I'd like to test it against something "real".

Lucky for all of us, there's actually an easy way out!

## Example

Consider this DTO:

### Customer.cs

```cs
internal class Customer
{
    [JsonPropertyName("first_name")]
    [Required]
    public string FirstName { get; set; } = null!;

    [JsonPropertyName("last_name")]
    [Required]
    public string LastName { get; set; } = null!;

    [JsonPropertyName("age")]
    [Range(18, 120)]
    public int Age { get; set; }

    [JsonPropertyName("date_of_birth")]
    [Required]
    public DateTime DateOfBirth { get; set; }

    [JsonPropertyName("address")]
    [Required]
    public string Address { get; set; } = null!;
    
    [JsonPropertyName("phone_number")]
    [Phone]
    public string? PhoneNumber { get; set; }

    [JsonPropertyName("email")]
    [EmailAddress]
    public string? Email { get; set; }
}
```

Filling this with data would be easy, because this specific DTO is not that large. But what if You needed multiple customers? It becomes quite an effort and is time consuming, sometimes even demotivating.

[Bogus](https://github.com/bchavez/Bogus) is an open-source fake data generator for .NET.

The usage is incredibly simple, to generate some fake data for these customers, let's create a fake data factory.

### FakeCustomerFactory.cs

```cs
internal static class FakeCustomerFactory
{
    public static Customer GenerateFakeCustomer()
    {
        var faker = CreateFaker();
        return faker.Generate();
    }

    public static List<Customer> GenerateFakeCustomers(int customerCount)
    {
        if (customerCount < 1)
            throw new ArgumentOutOfRangeException(nameof(customerCount), "Customer count must be positive.");
        
        return Enumerable.Range(0, customerCount)
            .Select(_ => GenerateFakeCustomer())
            .ToList();
    }

    private static Faker<Customer> CreateFaker(int minAge = 18, int maxAge = 80)
    {
        return new Faker<Customer>()
            .RuleFor(c => c.FirstName, f => f.Person.FirstName)
            .RuleFor(c => c.LastName, f => f.Person.LastName)
            .RuleFor(c => c.Age, f => f.Random.Int(minAge, maxAge))
            .RuleFor(c => c.DateOfBirth, (f, c) => f.Date.Past(c.Age))
            .RuleFor(c => c.Address, f => f.Address.FullAddress())
            .RuleFor(c => c.PhoneNumber, f => f.Phone.PhoneNumber("+420 ### ### ###"))
            .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.FirstName, c.LastName));
    }
}
```

This will enable us to create one or more fake customers. The faker could be injected into the factory, but for the sake of simplicity of this example, let's just create the object in a private method.

### Wait, what just happened?

Bogus holds a lot of fake data for different categories (customers, products etc.), and it is inspired by FluentValidation code style for creating fake objects. You simply set the rules for the data and You're good to go! But that's not the only way. Feel free to explore through Bogus repository to find out more ways and data to work with.

### Create the fake data

Creating the fake data now is super easy:

### Program.cs

```cs
var fakeCustomer = FakeCustomerFactory.GenerateFakeCustomer(); // 1 customer
var fakeCustomers = FakeCustomerFactory.GenerateFakeCustomers(5); // multiple customers
```

Great, running a debug on these data shows that the data are really there:

<a href="/assets/img/2023-08-14-testing-with-fake-data/fake_customers.png">
    <img 
        src="/assets/img/2023-08-14-testing-with-fake-data/fake_customers.png" 
        alt="Fake customers"
    >
</a>

## Is that it?!

Yes... and no! We wanted to test it against something real, right? Sending a POST with this data, or something... But why go through all the hassle of running Your own endpoint?

There's also another way.

### webhook.site

Using [webhook.site](https://webhook.site/) is a great way to test Your app communication and visualize data within a HTTP representation.

Webhook.site gives You an endpoint You could use and that's it, You just send a HTTP request from Your application:

### Program.cs

```cs
var fakeCustomers = FakeCustomerFactory.GenerateFakeCustomers(5);

const string endpoint = "https://webhook.site/{endpoint-id}";

using var client = new HttpClient();
var json = JsonSerializer.Serialize(fakeCustomers);
var content = new StringContent(json, Encoding.UTF8, "application/json");
        
client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

try
{
    var response = await client.PostAsync(endpoint, content);
    response.EnsureSuccessStatusCode();

    Console.WriteLine("HTTP POST request successful!");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Error sending HTTP POST request: {ex.Message}");
}
```

After sending the request, webhook.site will catch the request and show everything that it received:

<a href="/assets/img/2023-08-14-testing-with-fake-data/webhook_result.png">
    <img 
        src="/assets/img/2023-08-14-testing-with-fake-data/webhook_result.png" 
        alt="Webhook result"
    >
</a>

## That's it!

Keep in mind this is just a simple example of how You could utilize this simple tooling for testing Your own code, validations, creating fake data etc.