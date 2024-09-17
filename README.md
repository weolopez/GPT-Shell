# GPT-Shell

GPT-Shell is an OpenAI based chat-bot that is similar to OpenAI's ChatGPT (https://chat.openai.com/). It allows users to converse with a virtual companion. It uses nodejs and typescript, as well as modern yarn, to create a seamless conversation experience.

https://user-images.githubusercontent.com/108406948/207628213-c8bebc1b-ce72-45ba-907f-e0d6f51c27cd.mp4


## Try it out

You can try the bot on the official Discord server:

[![](https://dcbadge.vercel.app/api/server/TruuVEBmcC)](https://discord.gg/TruuVEBmcC)

## Usage

Set up a discord bot and add it to your server.

Follow the setup instructions below.

To interact with GPT-Shell, users can:
- Use the `/chat-gpt` command to start a conversation with the bot
- Ping the bot in a channel it's in
- Message the bot directly

The `/chat-gpt` command will start a new conversation thread, and whenever the user types something, the bot will respond.

The bot is able to handle multiple conversations at once,
so you can start as many conversations as you like.

## Setup

To use GPT-Shell, you will need to:
- Clone the project
- Run `yarn install`

You can also fork the replit:

https://replit.com/@Ephemeros/GPT-Shell

Set up the environment variables as described below.

Then to start a development environment, run `yarn dev`. To build and run, run `yarn build` and then `yarn start`.

## Environment Variables

The following environment variables are required for GPT-Shell to work properly.

MongoDB:
- MONGODB_URI: The MongoDB connection string. Should look something like this: `mongodb+srv://<username>:<password><cluster>.<something>.mongodb.net/?retryWrites=true&w=majority`
- DB_NAME: The name of the collection in MongoDB

Bot Token:
- BOT_TOKEN: The Discord bot token
  - You can get it by following https://discord.com/developers/applications then selecting your app and then selecting "Bot".

Commands:
- COMMAND_NAME: The name of the chat-gpt command
- API_KEY_COMMAND_NAME: The name of the chat-gpt-openai-api-key command (if you want users to add the bot to their server)

OpenAI API Key:
- OPENAI_API_KEY: The API key for your OpenAI account

Discord Server Details:
- MAIN_SERVER_ID: The Discord server where the bot lives
- LOG_CHANNEL_ID: (Optional) The channel for log messages
- MAIN_SERVER_INVITE: (Optional) The ideally non-expiring server invite link (if you want users to add the bot to their server)


You can set the environment variables in any way you like, or place an `.env.local` file at the root of your project,
that looks like this:

```
# MongoDB:
MONGODB_URI=
DB_NAME=

# Bot Token:
BOT_TOKEN=

# Commands:
COMMAND_NAME=chat-gpt
# API_KEY_COMMAND_NAME=chat-gpt-openai-api-key # enable this if you want the bot to be installed in other servers.

# OpenAI API Key:
OPENAI_API_KEY=

# Discord Server Details:
MAIN_SERVER_ID=
LOG_CHANNEL_ID=
MAIN_SERVER_INVITE=
```

## Contributions

We welcome contributions to GPT-Shell. If you have an idea for a new feature or have found a bug, please open an issue on GitHub. We would also appreciate any sponsorships or donations. You can sponsor us through our GitHub sponsor page [here](https://github.com/sponsors/firtoz).

## License

GPT-Shell is released under the MIT License.


# Set the buffer file path
BUFFER_FILE="$HOME/.command_errors_buffer"

# Ensure the buffer file exists
touch "$BUFFER_FILE"

# Prevent multiple redirections
if [[ -z "$ZSH_ERROR_CAPTURE_SETUP" ]]; then
    # Redirect stderr to buffer file while keeping stderr output to terminal
    exec 2> >(tee -a "$BUFFER_FILE" >&2)
    export ZSH_ERROR_CAPTURE_SETUP=1
fi

# Define the askai function
askai() {
    # Check for OpenAI API key
    if [[ -z "$OPENAI_API_KEY" ]]; then
        echo "Error: OPENAI_API_KEY environment variable is not set."
        return 1
    fi

    # Check if the buffer file exists and is not empty
    if [[ ! -s "$BUFFER_FILE" ]]; then
        echo "No errors to report."
        return 0
    fi

    # Read the buffer file content
    ERROR_CONTENT=$(cat "$BUFFER_FILE")

    # Prepare the JSON payload using jq for proper escaping
    PAYLOAD=$(jq -n \
        --arg model "gpt-4o" \
        --arg system_content "You are an expert zsh developer and system admin. Analyze this command line output from the terminal of an M1 MacBook Air and provide advice." \
        --arg user_content "$ERROR_CONTENT" \
        '{model: $model, messages: [{role: "system", content: $system_content}, {role: "user", content: $user_content}]}' )

    # Send the request
    RESPONSE=$(curl -sS -X POST "https://api.openai.com/v1/chat/completions" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $OPENAI_API_KEY" \
        -d "$PAYLOAD")

    # Check for errors in the response
    if [[ $? -ne 0 ]]; then
        echo "Error: Failed to contact OpenAI API."
        return 1
    fi

    # Check for errors in the API response
    API_ERROR=$(echo "$RESPONSE" | jq -r '.error.message' 2>/dev/null)

    if [[ "$API_ERROR" != "null" ]]; then
        echo "Error from OpenAI API: $API_ERROR"
        return 1
    fi

    # Extract the assistant's reply
    REPLY=$(echo "$RESPONSE" | jq -r '.choices[0].message.content' 2>/dev/null)

    if [[ -z "$REPLY" || "$REPLY" == "null" ]]; then
        echo "Error: Invalid response from OpenAI API."
        echo "Response from API:"
        echo "$RESPONSE"
        return 1
    fi

    # Print the assistant's reply
    echo -e "$REPLY"

    # Clear the buffer file
    > "$BUFFER_FILE"
}

