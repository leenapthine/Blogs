# Web Performance using CDNs, Async, and Defer

**Author**: [Lee Napthine](/team/lee-napthine)  
**Last updated:** 2025-02-01

**Originally published at:** [https://arcsoft.uvic.ca/log/2024-11-30-web-performance-cdns-async-defer/](https://arcsoft.uvic.ca/log/2024-11-30-web-performance-cdns-async-defer/)

On the ZooDB project I was tasked with looking into the case for and against using Content Delivery Networks (CDNs) rather than the locally hosted frameworks we had currently set up. A decision was made to try them along with their `async` and `defer` script attributes to determine whether web performance increased and in which areas.

## Content Delivery Networks

A Content Delivery Network is a system of distributed servers that deliver content to users based on geographical location. [Cloudflare](https://www.cloudflare.com/), known for its free tier, has extra security features and a database of scripts for most versions of available programs. We might also consider servers that specialize in libraries like JavaScript ([jsDelivr](https://www.jsdelivr.com/) or [cdnJS](https://cdnjs.com/)) or Bootstrap ([BootstrapCDN](https://www.bootstrapcdn.com/)).

### Why use a CDN?

- **Faster Load Times:** Resources are fetched from servers geographically closer to the user. This is important for caching data such as images and tables, as CDNs will send the data to nearby servers where it can be retrieved often and quickly. If the site is host to a worldwide audience, use CDNs.
- **Reduced Load on Local Server:** CDNs offload traffic from your origin server.
- **Improved Reliability:** Because CDNs are hosted by multiple servers, server failure is virtually never a problem. This is important if you run web based platforms that cannot risk going down for even short periods of time.
- **Smaller Size for Deployment:** CDNs allow you to reference hosted libraries (like Bootstrap, JQuery, etc.) instead of packaging them into your application. This reduces the size of your program or container significantly, particularly important when using tools like Docker which benefit from faster build times and lower storage requirements.

### When not to use CDNs

- **Customization:** If you need to make changes to the core code you will want the changes hosted locally. In this case, CDNs may not be what you need.
- **Blocked Regions:** I have read of cases where customers or regions have certain CDN providers blocked, this will lead to errors rendering the site. An example is Google Cloud CDN which was previously [blocked in China](https://edjiang.com/how-googles-cdn-prevents-your-site-from-loading-in-china-67504845cd04). Here is it important to know your target audience.
- **Offline Use:** If your application needs to run in places without a reliable internet connection, bundling libraries locally ensures they are available.

In my case, I designed a hybrid method which used fallback options for each CDN script. If any CDN link failed to load it would load from a locally hosted version of the library. Ultimately, we decided not to go this route, as it was deemed unnecessary and the team valued the smaller size when deploying the program.

## Using `async` and `defer` for Scripts

When using CDNs you will want to consider using the performance attributes `async` or `defer`.

### `async`

```html
<script src="demo_async.js" async></script>
```

Scripts with `async` load and execute as soon as they are ready, and do not wait for the rest of the page to finish parsing. Because the order that it loads is not guaranteed, developers must be careful where they use this as it can lead to errors if scripts depend on each other. I had issues using the async attribute. Small changes in the CSS styles were noticable as a result of scripts loading at incorrect times. It is best to use it for scripts with low dependencies like analytics or widgets.

### `defer`

```html
<script src="demo_defer.js" defer></script>
```

The `defer` attribute tells the browser to load the script after the initial page load has completed. Developers will want to use this for the bulk of big frameworks or libraries that have many dependencies such as JQuery and Bootstrap. Defer guarantees scripts will execute in the order in which they appear.

Often some combination of `async` and `defer` will lead to the optimal performance.

| Attribute | Loading Time           | Execution Order | Use Case                                         |
| --------- | ---------------------- | --------------- | ------------------------------------------------ |
| async     | As soon as it’s ready  | Not guaranteed  | Non-critical components with fewer dependencies  |
| defer     | After HTML parsing     | Guaranteed      | Critical components with many dependencies       |
| Neither   | As soon as encountered | Guaranteed      | Critical scripts that modify the DOM immediately |

## Testing Performance Improvements

After choosing to use CDNs and optimizing them with the chosen attribute, you will need to test your site’s performance and make sure the changes are an improvement. Certain factors, such as geographical location, will be hard to test for, as you would have likely been testing on a local nearby server prior to choosing a CDN script.

For my purposes I chose to use Google Chrome DevTools. Its widespread use makes it the ideal test candidate. Right click in any Chrome web browser and select the inspector.

Here are the metrics to test for:

1. **Largest Contentful Paint (LCP)**
   - Measures the time it takes for the largest visible content (like an image or a header) to load.
   - Under 2.5 seconds is considered good.
2. **Cumulative Layout Shift (CLS)**
   - Measures visible stability by looking at how long it takes for the layout to stop shifting around.
   - Aim for a low score of below 0.1 seconds here.
3. **First Contentful Paint (FCP)**
   - Tracks the time for the first content to render.
   - Time for this can vary, but lower is better, obviously.
4. **Time to Interactive (TTI)**
   - Measures when the site becomes fully interactive.
   - A good metric for optimizing user engagement.

Make sure you have tested your loading metrics prior to adding CDNs and performance attributes. After you have added them in, re-run performance tests and measure the improvements. Make sure to repeatedly clear your cache to get proper measurements of when a user first navigates to your site. Keep in mind that testing caching may not reflect the real-world benefits if you are close to the server of origin.

## Conclusion

Hopefully documenting my process in getting the best performance from ZooDB will be helpful to others. By implementing these methods and testing their impact on loading speeds and deployment size I think developers can boost website performance and improve user experience.
