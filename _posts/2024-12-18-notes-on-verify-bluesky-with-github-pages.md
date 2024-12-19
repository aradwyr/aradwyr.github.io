---
title: "Notes on: Verifying a Bluesky Account with Github Pages"
---

Assuming you already have a `<username>.github.io` Github site setup, the process to link to your Bluesky account is fairly straightforward. 

In your Bluesky account, navigate to **Settings** > **Account** > **Handle** > select `I have my own domain` then: 
- Domain: `<username>.github.io`  
- Choose the `No DNS Panel` option
- Note/copy the value listed under `contains the following` for the `https://<github-url>/.well-known/atproto-did` file path; it will look something like [did:plc:zzzzz](https://github.com/aradwyr/aradwyr.github.io/blob/main/.well-known/atproto-did). 
- Naturally, don't select `Verify Text File` until pushing changes listed below to Github in your public `<username>.github.io` repo first. 


Meanwhile, in your root directory of your `<username>.github.io` repository, you'll add the following: 

- In the `_config.yml` file (if one doesn't already exist): 
    ```
    include: [".well-known"]
    ```

- In the `.well-known/atproto-did` file path: 
    ```
    did:plc:<replace-with-str>
    ```


### References
- For more background on [Github Pages](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site) custom domains
- Bluesky documentation: [How to verify your Bluesky account](https://bsky.social/about/blog/4-28-2023-domain-handle-tutorial)