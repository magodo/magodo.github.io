---
layout: "post"
title: "Use Claude Code with Github Copilot Subscription via LiteLLM"
categories: "blog"
tags: ['ai']
published: true
comments: true
script: [post.js]
excerpted: |
    This post describes how to setup `litellm` to use `claude` via Github Copilot subscription.
---

* TOC
{:toc}

My organization grants us Github Copilot subscription with all the LLM models access, while we can only use it through Github Copilot ecosystem. It would be better if one can use Claude Code while directs all LLM requests to github copilot LLMs (hence free of billing). This is possible by using [`litellm`](https://docs.litellm.ai/docs/). This post will record my setup to make this working.

# 1. Setup `litellm` 

## 1.1 Install `litellm[proxy]`

> LiteLLM is an open-source library that gives you a single, unified interface to call 100+ LLMs — OpenAI, Anthropic, Vertex AI, Bedrock, and more — using the OpenAI forma

We are not going to use it as a library, but to use its *self-hostsed* LLM Gateway (Proxy). I use `uv` to install it:

```
uv tool install 'litellm[proxy]'
```

> 💡 In Python packaging, the brackets `[...]` denote an extra. While the base `litellm` package is a lightweight library meant to be used inside your Python code, the `proxy` extra adds the heavy-duty components needed to run a standalone server.

## 1.2 Setup `config.yaml`

The official website has a page describing how to setup the Github Copilot provider [here](https://docs.litellm.ai/docs/providers/github_copilot).

The example `config.yaml` uses some relative old models, which are not closely matching claude code's expectation as well. Instead, I used the mitmproxy to inspec the API requests sent from copilot VSCode extension and found a request to `https://api.githubcopilot.com/models`, which return all the avaialble models in Github Copilot. Then I pick the latest Anthropic models:

```yaml
model_list:
  - model_name: sonnet
    litellm_params:
      model: github_copilot/claude-sonnet-4.6

  - model_name: haiku
    litellm_params:
      model: github_copilot/claude-haiku-4.5

  - model_name: opus
    litellm_params:
      model: github_copilot/claude-opus-4.6-1m

litellm_settings:
  # Claude Code (the client) will send a "thinking" parameter to enable reasoning, but LiteLLM knows that the GitHub 
  # Copilot provider doesn't officially support that specific parameter in its API schema yet.
  # To fix this, tell LiteLLM to "ignore" or "drop" parameters it doesn't recognize so it can proceed with the request.
  drop_params: True

  # The master key is sourced from env var "LITELLM_MASTER_KEY" at the session starts the litellm.
  # If this is not set, there is no Authorization header needed for the client request.
  #   master_key: os.environ/LITELLM_MASTER_KEY
```

Apart from the `model_list` setting, I've two `litellm_settings` settings, which are explained by the comments.

## 1.3 Run `litellm`

With above, one can run `litellm` via:

```
litellm --config config.yaml
```

In case you want to inspect any outgoing request (e.g. to github copilot), export the following env var before start (given you have `mitmproxy` installed):

```
export SSL_CERT_FILE=~/.mitmproxy/mitmproxy-ca-cert.pem
```

# 2. Run Claude Code

## 1.1 Install `claude`

I install claude code via `yay` in archlinux. 

## 1.2 Setup and Run `claude`

The official page for running `claude` via `litellm` can be found at [here](https://docs.litellm.ai/docs/tutorials/claude_responses_api#4-configure-claude-code). This is not very clear especially it is confusing about which env vars to set for which app.

Actually, what you really need is:

```shell
# Set the URL endpoint to the litellm
export ANTHROPIC_BASE_URL="http://0.0.0.0:4000"

# Set the auth token, which is necessary to bypass the login process.
# The value doesn't matter unless `litellm`'s `litellm_settings.master_key` is specified.
export ANTHROPIC_AUTH_TOKEN="foo"

# Set the default models to the models available from your llm provider (setting in the litellm config.yaml)
# This step can be omitted via using `--model <model>`, whilst it will cause some error as one prompt can 
# tirgger multiple completion requests using different models.
export ANTHROPIC_DEFAULT_SONNET_MODEL=sonnet
export ANTHROPIC_DEFAULT_HAIKU_MODEL=haiku
export ANTHROPIC_DEFAULT_OPUS_MODEL=opus
```

To persist them, one can add them into the `~/.claude/settings.json`:

```json
{
  //...
  "env": {
    "ANTHROPIC_BASE_URL": "http://0.0.0.0:4000",
    "ANTHROPIC_AUTH_TOKEN": "foo",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "sonnet",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "haiku",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "opus"
  }
}
```

Next, you can simply run `claude` without any additional options.
