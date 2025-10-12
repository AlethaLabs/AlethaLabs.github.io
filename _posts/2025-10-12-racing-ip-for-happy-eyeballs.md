---
layout: post
title: "Racing IP for Happy Eyeballs p.1 of 2"
date: 2025-10-12 08:35:00 +0000
categories: [networking, rust]
---

# Connection Races for Happy Eyes p.1 of 2

## Part One of Two

This is going to be a two-part series and introduction into the algorithm 'Happy Eyeballs'. In today's post we will be explaining what 'Happy Eyeballs' is, the steps to create the algorithm, and then have some fun programming the first step - which is: Performing asynchronous DNS queries of a given host and displaying the results. For this we will be using rust, and in part two we will be jumping straight into the code and implementing the core functionality of 'Happy Eyeballs'. If you would like to view and test the full implementation before part two of this series, you can check out my [GitHub here](https://github.com/AlethaLabs/happy-eyes)

## What is 'Happy Eyeballs'?

Happy Eyeballs was a new proposed standard by the Internet Engineering Task Force (IETF) in mid-2012. Labeled - [RFC 6555](https://datatracker.ietf.org/doc/html/rfc6555) - it addressed the developing problem of IPv6 vs IPv4 connection from a client (you) to a DNS server. If you were a dual-stack client, meaning your system could use either IPv6 and IPv4, and you were trying to connect to a server, whose IPv6 path and protocol was not working, you would experience significant connection delay. This new proposed standard set an algorithm in place that would help mitigate this issue with the ever expanding release of IPv6, thus the name ***"Happy Eyeballs"*** was bestowed.

Since the creation of Happy Eyeballs, there have been new proposed standards to improve on the algorithm and include newer protocols, such as Transport Layer Security (TLS). [Happy Eyeballs v2 - RFC 8305](https://datatracker.ietf.org/doc/html/rfc8305) - was proposed in December of 2017, and has been the standard ever since. Recently this year - 2025 - there has been a draft paper for version 3 of this algorithm that incorporates even newer protocols such as Quick UDP Internet Connection (QUIC). You can find the latest paper [here](https://datatracker.ietf.org/doc/draft-ietf-happy-happyeyeballs-v3/), but for the purpose of today's blog, we will be focusing primarily on Happy Eyeballs **v2**.

### The Algorithm

The algorithm is beautifully conceptualized and the work behind the scenes of something as grand as the internet never ceases to amaze me. If any terminology is confusing to you throughout the rest of this blog, I suggest taking a quick google detour as I will not be explaining all details, such as the difference between AAAA and A records.

The algorithm goes as follows:

1.  **Initiate asynchronous DNS queries**
    
    - AAAA and A records should be made as soon as possible after one another
    - DNS resolution should be treated asynchronously
    - If a positive AAAA response is received first, start connection immediately
    - If a positive A response is received before the AAAA response, the client should wait 50ms to be sure no AAAA response is lagging behind before proceeding with connection. This is to ensure the prioritization of IPv6 over IPv4 - this delay is called the - "Resolution Delay"
2.  **Build and sort the destination address list in spec with [RFC 6724 - Section 6](https://datatracker.ietf.org/doc/html/rfc6724)**
    
3.  **Interleave address families (avoid long same-family runs)**
    
4.  **Start connection attempts, one at a time, staggered**
    
5.  **Once one connection succeeds, cancel the others**
    

The steps above are obviously simplified and the RFC goes into things such as:

- How to react to DNS answers that change while you are racing IPv4 and IPv6
- How to handle IPv6-only networks
- Security Considerations
- Limitations
- IP Address Literals
- etc...

However, we will just be covering the core implementation of the Happy Eyeballs algorithm and save the nuances for another time, now it's time to code.

## Rust for Happy Eyeballs

We will implement the Happy Eyeballs algorithm using the rust programming language. It is fairly short code in the grand scheme of things, but there is still a-lot to cover, so buckle up! It's time to get rusty!

### Dependencies

As with any project in rust, unless you want to reinvent the wheel, you will have some dependencies and these listed below are what we will be using today.

![d296e4c738eb56be2c999a69a1f10d8a.png](/assets/images/posts/d296e4c738eb56be2c999a69a1f10d8a.png)

As we move along, we will dive deeper into these crates and I will explain why they are needed in context. These are all amazing projects with many brilliant individuals working on them day and night. Again, it never ceases to amaze me how much work goes into the world of tech, especially opensource tooling! Now lets get to the first step of writing this algorithm.

### Initiate Asynchronous Queries

Lets take a look at some of the imports we will be using to perform asynchronous queries and DNS resolution:

![7ef76ac5dbddfdc2f98fce1703cf7731.png](/assets/images/posts/7ef76ac5dbddfdc2f98fce1703cf7731.png)

- We have the standard networking and time crate
- We also have a really cool crate called [Hickory Resolver](https://docs.rs/hickory-resolver/latest/hickory_resolver/index.html) that can perform asynchronous, recursive queries to lookup domain names
- And of course, in rust fashion, we have a basic error type to handle whatever we may run into and clean up function signatures

Now we can move onto the function signature and set up some variables:

![18aabcb5fc8517605848e100fcdf0b9f.png](/assets/images/posts/18aabcb5fc8517605848e100fcdf0b9f.png)

Here we have the beginning of an async function - `resolve_dns` - that takes in host and port as parameters, this function needs to return - `Vec<SocketAddr>` - from the standard library that holds both the IP and its respective port, along with the time in milliseconds it took to resolve. Fun fact: an unsigned 32 bit integer can hold about 49 days of milliseconds!

The actual function starts out by setting up some variables:

- **`dns_start`**
    - Creates an instant from the standard time crate which allows for precise time measurement

- **`resolver`**
    - This variable holds the core of our DNS resolution
    - From hickory_resolver we use `TokioResolver::builder_tokio()` to create a resolver consistent with our systems configuration
    - `builder_tokio()` is a convenience function that simplifies code
    - In Linux it will use your /etc/resolv.conf file to build from

- **`qa_start`**    
    - *'qa'* stands for *'Quad A'* to simplify writing aaaa_ or ipv6_... for every instance
    - We create an instant in time once again for precise measurement

- **`qa_future`**
    - Using our resolver we created earlier we can now query a given host with our IPv6 address
    - `Box::pin()` allows us to create a heap allocated pinned future, which will help us use `tokio::select!`
    - We will explain more about this in the next section

- **`a_start/a_future`**    
    - These do the same thing as the previous two variables, but for IPv4

- **`qa_completed/a_completed`**    
    - We set these to false to act as a *guard* when using `tokio::select!`
    - These variables will help us determine if we need to wait for the 50ms *Resolution Delay* before connecting to the host
    - More on this in the next section
***
**Now we can get to some more complex code, but first:**

- **As mentioned above, the Happy Eyeballs algorithm states:**
    - If a positive AAAA response is received first, start connection immediately
    - If a positive A response is received before the AAAA response, the client should wait 50ms to be sure no AAAA response is lagging behind before proceeding with connection. This is to ensure the prioritization of IPv6 over IPv4 - this delay is called the - "Resolution Delay"

We can do this while still retaining concurrency by using:

- **`tokio::select!`**
    - You can find the documentation for this macro [here](https://docs.rs/tokio/latest/tokio/macro.select.html)
    - It waits on multiple concurrent branches, returning when the first branch completes, cancelling the remaining branches
    - It is a complex yet brilliant protocol

![5efd5292acc11338fbe9abc85e7b1ab8.png](/assets/images/posts/5efd5292acc11338fbe9abc85e7b1ab8.png)

Lets take a closer look at what is happening here:

- **`qa_result`**
    
    - This first line sets up our conditional asynchronous operation that waits for the IPv6 queries to complete
    - If we remember from above, we preset `qa_completed` to false, so here we are saying in plain English:
        - If not completed, enable this branch, run this code and put the results in `qa_result`
    - This then starts the first branch of our concurrent operation
- **Why set `{ qa_completed = true; }`?**
    
    - Here we set this to true so the precondition `!qa_completed` now evaluates to false
    - This signals to `tokio::select!` that this branch should be disabled and no longer polled
    - We set this value immediately to prevent the same branch from running multiple times
    - We need to use `#[allow(unused_assignments)]`because the compiler is being confused by the tokio runtime
    - A little confusing, but we got there eventually!
- **`match qa_result {}`**
    
    - Here we have a fairly straight forward match expression
    - If there aren't any errors, we proceed with IPv6 lookup and iterate over the results and format the output

**Within spec of this algorithm, we must make sure to still collect the A records to race connection attempts later**

- **`if !a_completed{}`**
    - This is pretty much the same code as above, just slightly simplified
    - All we are doing is looking up the A records, iterating over and formatting them, then displaying the output with it's respective time measurement

**Now, below we have pretty much the same code as the previous block with a few caveats:**  
- We are now handling the situation where *"IPv4 lookup"* has finished first  
- As stated above, if this situation happens, we must wait no more than 50ms for IPv6 to try and finish  
- This is to ensure we are still maintaining priority of IPv6 over IPv4, but not too much, that we cause significant delay  
- We accomplish this by using `tokio::time::sleep()` on line **78**

![47da1941bac8352bab40c4395705d7fc.png](/assets/images/posts/47da1941bac8352bab40c4395705d7fc.png)

**Wrapping up `tokio::select!`**

- Now that we have done everything we can to ensure IPv6 supremacy, we can process the IPv4 results
    - We do that again, with a simple match statement
    - If no error is present, we iterate over, count the results, measure the time in ms and display the output

![e0676aa4306bc233fc7d7dc076ed892a.png](/assets/images/posts/e0676aa4306bc233fc7d7dc076ed892a.png)

***
**That is the end of the `tokio::select!` block**  
To wrap up this function, we just need to print the results and check for any errors:

![bd3438eec1b57e73da02fd2ff2fd6e83.png](/assets/images/posts/bd3438eec1b57e73da02fd2ff2fd6e83.png)

To test this - all you need to do is add this function we have created into your main function, but first make sure that you are using the tokio runtime, which we will go into more in part two of this series:

```rust
#[tokio::main]
async fn main() -> Result<()> {
    resolve_dns("example.com", 443).await()?;

    Ok(())
}
```

Then run it in from your terminal:  
`cargo run`

The output will look like this:

![d039626a065ad2269af03ddf4fc23ae5.png](/assets/images/posts/d039626a065ad2269af03ddf4fc23ae5.png)

A couple of notes:

- You might get different results, as I forgot to delete my systems web-cache before running this
- Also sometimes, IPv4 will finish first, which is why we made the code in a particular manner
- Below is the output of running this a second time, immediately after the first and you can see that our A query completed before our AAAA query

![1716dd72ab12ee134c515764664e2105.png](/assets/images/posts/1716dd72ab12ee134c515764664e2105.png)

***
## Closing Thoughts
Well that is it for the beginning part of our implementation of the Happy Eyeballs algorithm written in rust. I have had a lot of fun during this project and have learned so much about asynchronous programming. If you want to check out the full implementation, please visit my [GitHub](https://github.com/AlethaLabs/happy-eyes)

Have a good day, thanks for reading, and see you for part two!
\- Chris

[LinkedIn](www.linkedin.com/in/christopher-piccus-b287a2386)
[GitHub](https://github.com/AlethaLabs/happy-eyes)