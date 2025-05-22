**LLM-proxy-application** is an LLM intermediary that sits between your software and the LLM provider's API. It captures requests and responses, enabling functionalities such as caching, request throttling, and key administration.

**LLM-proxy-application** incorporates the following capabilities:

**Core functionalities:**

- **Model Redirection:** Steer requests to various LLM providers (currently OpenAI and Mistral AI) according to the specified model.
- **Failover:** Automatically transition to an alternative provider if the main provider encounters issues, ensuring high uptime.
- **Authentication:** Secure entry using virtual API keys, enabling you to manage and regulate access for different users/applications.
- **Request Throttling:** Enforce per-user request limits based on both requests per minute and tokens per minute, deterring misuse and controlling resource consumption.
- **Expenditure Monitoring:** Compute and monitor the cost associated with each request, offering visibility into your LLM usage and expenses.
- **Database-Driven Configuration:** Retain all configurations (models, providers, user keys, rate limits) in a PostgreSQL database, simplifying the management and updating of settings without code alterations.
- **Expandable:** Architected for straightforward extension to include support for more LLM providers.
- **Request and Response Harmonization:** Supports OpenAI and Mistral AI providers, with requests and responses adhering to the OpenAI format for uniformity.

## Preconditions

Before you commence, ensure you have the subsequent items installed:

- **Node.js:** (version 16 or newer suggested). Obtain from [https://nodejs.org/](https://nodejs.org/).
- **npm:** (Node Package Manager) Typically included with Node.js.
- **PostgreSQL:** A running PostgreSQL server is necessary. Obtain from [https://www.postgresql.org/download/](https://www.postgresql.org/download/). Adhere to the instructions for your operating system.
- **`psql`:** The PostgreSQL command-line utility (usually installed alongside PostgreSQL).
- **`openssl`:** For creating encryption keys (usually pre-installed on Linux/macOS; for Windows, it's often part of Git). Verify with `openssl version`.
- **`git`:** To duplicate the repository.
- **`curl`:** For API testing (or alternatives like Postman, Insomnia, etc.).

## Installation and Configuration

1.  **Clone the repository:**

    ```bash
    git clone <your-repository-url-here>
    cd <your-repository-directory-here>
    ```

2.  **Install dependencies:**

    ```bash
    npm install
    ```

3.  **Configure environment variables (ESSENTIAL):**

    You _must_ define two environment variables for encryption: `ENCRYPTION_KEY` and `IV`. _Do not overlook this step_.

    - **Generate random keys:** Employ `openssl` to create strong, random keys:

      ```bash
      export ENCRYPTION_KEY=$(openssl rand -hex 32)
      export IV=$(openssl rand -hex 16)
      ```

    - **Confirm:** IMMEDIATELY confirm that the variables are defined:

      ```bash
      echo $ENCRYPTION_KEY  # Should display 64 hexadecimal characters
      echo $IV              # Should display 32 hexadecimal characters
      ```

    - **(.env file - DEVELOPMENT PURPOSES ONLY):** For _development purposes solely_, you can establish a `.env` file in the project root:

      ```
      ENCRYPTION_KEY=your_64_char_hex_key_here
      IV=your_32_char_hex_iv_here
      ```

      Substitute `your_64_char_hex_key` and `your_32_char_hex_iv` with _your_ generated keys. If utilizing a `.env` file, ensure the _initial_ line of `app.js` is `require('dotenv').config();` and that you've installed `dotenv` (`npm install dotenv`). **Never commit the .env file to version control.**

    - **Critical:** For production environments, utilize your operating system or hosting provider's recommended method for securely setting environment variables.

4.  **Configure PostgreSQL:**

    - **Ensure PostgreSQL is operational.**

    - **Create database and user:** Connect to PostgreSQL as the `postgres` superuser (or another administrative user):

      - **Linux/macOS:** `sudo -u postgres psql`
      - **Windows:** (Assuming `psql` is in your system's PATH) `psql -U postgres`

      Execute these SQL statements in `psql`:

      ```sql
      CREATE DATABASE llm_proxy_db;
      CREATE USER llm_proxy_user WITH PASSWORD 'a_very_strong_password_here';  -- ***ALWAYS USE A STRONG PASSWORD IN PRODUCTION***
      GRANT ALL PRIVILEGES ON DATABASE llm_proxy_db TO llm_proxy_user;
      \q  -- Exit psql
      ```

      **SECURITY ADVISORY:** Using simplistic names like `llm_proxy_user` for username and `llm_proxy_db` for the database name with a weak password is for _development convenience only_. _Never replicate this in a production setting._

5.  **Initialize Sequelize:**

    ```bash
    npx sequelize-cli init
    node setup-config.js
    ```

    This step creates the `config`, `models`, `migrations`, and `seeders` directories.

6.  **Configure Sequelize (`config/config.json`):**

    - Open `config/config.json`.
    - Adjust the `development`, `test`, and `production` sections to align with your PostgreSQL settings. Confirm that `username`, `password`, `database`, `host`, and `dialect` are accurate (the values below are examples). Include the `schema` option.

    ```json
    {
      "development": {
        "username": "llm_proxy_user",
        "password": "your_dev_db_password",
        "database": "llm_proxy_db",
        "host": "127.0.0.1",
        "dialect": "postgres",
        "schema": "public"
      },
      "test": {
        "username": "llm_proxy_user",
        "password": "your_test_db_password",
        "database": "llm_proxy_db_test",
        "host": "127.0.0.1",
        "dialect": "postgres",
        "schema": "public"
      },
      "production": {
        "username": "your_prod_db_user",
        "password": "your_prod_db_password",
        "database": "llm_proxy_db_prod",
        "host": "your_prod_db_host",
        "dialect": "postgres",
        "schema": "public"
      }
    }
    ```

    - Also, open `models/index.js`, and find this line:

    ```javascript
    const config = require(__dirname + "/../config.json")[env];
    ```

    And change it to:

    ```javascript
    const config = require(__dirname + "/../config/config.json")[env]; // Corrected path assuming config.json is in config folder
    ```

    _(Self-correction: The original instructions for `models/index.js` had identical "before" and "after" lines and a potentially incorrect path. I've corrected the path to point to `../config/config.json` which is more standard with Sequelize's `sequelize-cli init` structure.)_

7.  **Create models and migrations:**

    If needed, execute these commands to generate model and migration templates:

    ```bash
    npx sequelize-cli model:generate --name User --attributes virtualKey:string,openaiKey:string,mistralKey:string,requestsPerMinute:integer,tokensPerMinute:integer,totalCost:float
    npx sequelize-cli model:generate --name LLMModel --attributes name:string,provider:string,fallback:string,inputCostPer1k:float,outputCostPer1k:float
    npx sequelize-cli model:generate --name Provider --attributes name:string,apiBase:string,apiVersion:string
    ```

    - **Adapt the generated files:** Meticulously replace the content of the generated model files (`models/user.js`, `models/llmmodel.js`, `models/provider.js`) and their corresponding migration files within the `migrations/` directory with the code structure intended for your application. Pay close attention to:
      - The `encryptKey` and `decryptKey` methods in `models/user.js`.
      - The `fallback` field in `models/llmmodel.js` (and its getter/setter methods).
      - The `tableName` option in each model to ensure the accurate table names are employed.
        _(Note: The original instruction "with the code provided in the previous responses" is retained as its specific content is unknown.)_

8.  **Execute migrations:**

    ```bash
    npx sequelize-cli db:migrate
    ```

    This action creates the tables in your database. Confirm with `psql` using `\dt`.

9.  **Execute seeders:**

    Run the subsequent command to execute all seeder files and populate your database with initial data:

    ```bash
    ./runseeders.sh
    ```

    This script will run all seeders via Sequelize CLI, populating your database with the required initial data.

## Launching the application

```bash
export ENCRYPTION_KEY=$(openssl rand -hex 32)  # If NOT using a .env file
export IV=$(openssl rand -hex 16)              # If NOT using a .env file
node app.js
```

You should observe a message like "LLM proxy server listening on port 3000. Connected to PostgreSQL via Sequelize".

## Verifying the API

Utilize curl (or Postman, Insomnia, etc.):

### 1. Create a virtual key:

```bash
curl -X POST http://localhost:3000/api/generate-virtual-key
```

This command returns a response similar to:

```json
{ "virtualKey": "a1b2c3d4-e5f6-7890-1234-567890abcdef" }
```

### 2. Store API Keys:

```bash
curl -X POST -H "Content-Type: application/json" -d '{
  "virtualKey": "YOUR_GENERATED_VIRTUAL_KEY",
  "openaiKey": "sk-your-test-openai-key",
  "mistralKey": "your-test-mistral-key"
}' http://localhost:3000/api/save-keys
```

Replace:

- `YOUR_GENERATED_VIRTUAL_KEY` with the key from step 1.
- `sk-your-test-openai-key` with a test OpenAI key (or an empty string "").
- `your-test-mistral-key` with a test Mistral key (or an empty string "").

### 3. Dispatch a chat completion request:

```bash
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer YOUR_VIRTUAL_KEY" -d '{
  "model": "gpt-3.5-turbo",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant."
    },
    {
      "role": "user",
      "content": "Translate the following English text to French: Hello, world!"
    }
  ]
}' http://localhost:3000/chat/completions
```

Substitute `YOUR_VIRTUAL_KEY` with your generated key.

### 4. Test failover mechanism:

Temporarily make your OpenAI key invalid in the database (using `psql`) and send another request. The proxy should then redirect to Mistral. Afterward, restore your valid OpenAI key.

### 5. Test request throttling:

- Requests Per Minute: Send more than 60 requests (or your configured lower threshold for easier testing) within a one-minute interval. You should receive `429 Too Many Requests` errors.
- Tokens Per Minute: Dispatch requests with very extensive prompts to surpass the token limit within a minute.

### 6. Confirm cost tracking: Connect to your database using `psql` and inspect the `totalCost` column in the `Users` table. It should be incrementing:

```sql
psql -U llm_proxy_user -d llm_proxy_db -h localhost
SELECT * FROM "Users";
```

### 7. Test error responses (should return 401 Unauthorized):

Invalid virtual key:

```bash
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer invalid_key" -d '{"model": "gpt-3.5-turbo", "messages": []}' http://localhost:3000/chat/completions
```

Missing Bearer token prefix: (Should yield 401 Unauthorized)

```bash
curl -X POST -H "Content-Type: application/json" -H "Authorization: your_virtual_key" -d '{"model": "gpt-3.5-turbo", "messages": []}' http://localhost:3000/chat/completions
```

Invalid model name: (Should yield 400 Bad Request)

```bash
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer YOUR_VIRTUAL_KEY" -d '{"model": "invalid-model", "messages": []}' http://localhost:3000/chat/completions
```

## Constraints and Future Enhancements

This configuration is intended for development. For production deployment:

- Never embed API keys or encryption keys directly in your code or `.env` file in a production environment. Employ environment variables or a dedicated key management service.
- Utilize HTTPS for secure communication.
- Implement thorough input validation.
- Consider robust authentication/authorization for your API endpoints.
- Periodically update all dependencies.

Potential areas for improvement:

- **Error Management:** Current error handling is rudimentary and requires enhancement.
- **Request Throttling:** This version uses in-memory rate limiting. Redis or a similar solution should be explored for production scalability.
- **Database:** The database setup currently creates a public schema by default; review schema and permissions for production.
