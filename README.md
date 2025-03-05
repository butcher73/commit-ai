# Gemini Commit Message Generator (Pre-commit Hook)

This script is a pre-commit Git hook that automatically generates commit messages using the Gemini AI API. It analyzes your staged changes, sends them to Gemini, and uses the generated message for your commit.

## Setup

1.  **Prerequisites:**
    * Git
    * `curl`
    * `jq`
    * Gemini API Key
    * A package manager like `pnpm`, `npm`, or `yarn` (for Husky installation)

2.  **Install Husky:**
    * Navigate to your project's root directory in your terminal.
    * Install Husky:

        ```bash
        pnpm add husky --save-dev
        ```

        or

        ```bash
        npm install husky --save-dev
        ```

        or

        ```bash
        yarn add husky --save-dev
        ```

3.  **Enable Git Hooks:**
    * Tell Git to use Husky's hooks:

        ```bash
        git config core.hooksPath .husky
        ```

4.  **Create the `pre-commit` Hook:**
    * Create a file named `pre-commit` inside the `.husky` directory.
    * Copy and paste the following script into the `pre-commit` file:

        ```bash
        #!/bin/bash

        # Load environment variables from .env file (if it exists)
        if [ -f .env ]; then
          export $(grep -v '^#' .env | xargs)
        fi

        # Get the diff of staged changes
        DIFF=$(git diff --staged)

        if [ -z "$DIFF" ]; then
          echo "No staged changes. Skipping commit message generation."
          exit 0
        fi

        # Base64 encode the diff
        BASE64_DIFF=$(echo -n "$DIFF" | base64)

        # Prepare the prompt for Gemini API
        PROMPT="Write a concise commit message (30-50 words) based on the following git diff (Base64 encoded):\n\n$BASE64_DIFF"

        # Call Gemini API
        GEMINI_API_KEY="$GEMINI_API_KEY" # Read API key from environment variable

        # Check if the API key is set
        if [ -z "$GEMINI_API_KEY" ]; then
            echo "Error: GEMINI_API_KEY environment variable is not set."
            exit 1
        fi

        GEMINI_API_ENDPOINT="[https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=$GEMINI_API_KEY](https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=$GEMINI_API_KEY)"

        API_RESPONSE=$(curl -s -X POST \
          -H "Content-Type: application/json" \
          -d '{
            "contents": [{
              "parts": [{"text": "'"$PROMPT"'"}]
            }],
            "generationConfig": {
              "temperature": 0.3,
              "maxOutputTokens": 150
            }
          }' \
          "$GEMINI_API_ENDPOINT")

        # Check for curl errors
        if [ $? -ne 0 ]; then
            echo "Error: curl command failed. Check your network connection and API endpoint."
            exit 1
        fi

        # Extract the commit message
        COMMIT_MESSAGE=$(echo "$API_RESPONSE" | jq -r '.candidates[0].content.parts[0].text')

        # Handle API errors
        if [ -z "$COMMIT_MESSAGE" ]; then
            echo "Error: Failed to generate commit message from API response."
            exit 1
        fi

        # Clean up the commit message
        COMMIT_MESSAGE=$(echo "$COMMIT_MESSAGE" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//')

        # Display the generated commit message
        echo "Generated commit message:"
        echo "$COMMIT_MESSAGE"
        echo ""

        # Use the generated commit message in the commit, bypassing the hook
        git commit --no-verify -m "$COMMIT_MESSAGE"

        echo "Commit complete."
        ```

    * Make the `pre-commit` file executable:

        ```bash
        chmod +x .husky/pre-commit
        ```

5.  **Set Your API Key:**
    * Create a `.env` file in the root of your project.
    * Add your Gemini API key to the `.env` file:

        ```
        GEMINI_API_KEY="YOUR_ACTUAL_API_KEY"
        ```

        Replace `"YOUR_ACTUAL_API_KEY"` with your actual API key.
    * **Important:** Add `.env` to your `.gitignore` file to prevent committing your API key.

## Usage

1.  **Stage Your Changes:**
    * Make changes to your project.
    * Stage the changes using `git add .` or `git add <file>`.

2.  **Commit Your Changes:**
    * Run `git commit`.
    * The pre-commit hook will automatically run:
        * It will analyze your staged changes.
        * It will send the changes to the Gemini API.
        * It will use the generated commit message for your commit.

3.  **Troubleshooting:**
    * If you encounter errors, check the terminal output for error messages.
    * Verify that your API key is correct and set in the `.env` file.
    * Ensure you have a stable internet connection.
    * Make sure `jq` and `curl` are installed.
    * If the commit message is not generated, check the API response (you can temporarily uncomment the `echo "Raw API Response:"` line to see it).

## Security

* **Never commit your API key directly to your repository.**
* Using environment variables and `.env` files is a safer approach.
* Always add `.env` to your `.gitignore` file.
* Be mindful of API usage and potential costs.