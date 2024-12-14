Certainly! Let's build an **advanced** Python setup for using Smartproxy's rotating proxies to send HTTP requests. This setup will include:

1. **Asynchronous Requests:** Utilize `asyncio` and `httpx` for concurrent HTTP requests.
2. **Robust Error Handling & Retries:** Implement retry logic with exponential backoff for handling transient failures.
3. **Logging:** Integrate logging to monitor proxy usage, errors, and request statuses.
4. **Configuration Management:** Use environment variables or configuration files for sensitive data like credentials.
5. **Proxy Pool Management:** Although Smartproxy handles rotation, we'll include mechanisms to handle proxy failures gracefully.
6. **Rate Limiting:** Prevent overwhelming target servers and avoid getting blocked.

### **1. Setup and Requirements**

#### **Install Required Libraries**

Ensure you have Python 3.7+ installed. Install the necessary libraries using `pip`:

```bash
pip install httpx asyncio tenacity python-dotenv
```

- **httpx:** For making asynchronous HTTP requests.
- **asyncio:** Python's built-in library for asynchronous programming.
- **tenacity:** For implementing retry logic with exponential backoff.
- **python-dotenv:** For managing environment variables securely.

### **2. Configuration Management**

Use a `.env` file to store sensitive information like Smartproxy credentials. This approach keeps your credentials secure and separate from your codebase.

#### **Create a `.env` File**

Create a file named `.env` in your project directory with the following content:

```dotenv
SMARTPROXY_USERNAME=your_username
SMARTPROXY_PASSWORD=your_password
SMARTPROXY_ENDPOINT=gate.smartproxy.com:7000  # Replace with your Smartproxy endpoint and port
REQUEST_TIMEOUT=10  # Timeout for each request in seconds
MAX_RETRIES=5  # Maximum number of retries for failed requests
CONCURRENT_REQUESTS=10  # Number of concurrent requests
TARGET_URL=http://httpbin.org/ip  # Example URL to fetch
```

> **Security Note:** Ensure that your `.env` file is included in `.gitignore` to prevent accidental commits of sensitive information.

#### **Load Configuration in Python**

```python
import os
from dotenv import load_dotenv

# Load environment variables from .env file
load_dotenv()

SMARTPROXY_USERNAME = os.getenv('SMARTPROXY_USERNAME')
SMARTPROXY_PASSWORD = os.getenv('SMARTPROXY_PASSWORD')
SMARTPROXY_ENDPOINT = os.getenv('SMARTPROXY_ENDPOINT', 'gate.smartproxy.com:7000')
REQUEST_TIMEOUT = int(os.getenv('REQUEST_TIMEOUT', 10))
MAX_RETRIES = int(os.getenv('MAX_RETRIES', 5))
CONCURRENT_REQUESTS = int(os.getenv('CONCURRENT_REQUESTS', 10))
TARGET_URL = os.getenv('TARGET_URL', 'http://httpbin.org/ip')
```

### **3. Advanced Python Implementation**

Below is the complete advanced Python script incorporating all the features mentioned:

```python
import os
import asyncio
import httpx
import logging
from dotenv import load_dotenv
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

# Load environment variables from .env file
load_dotenv()

# Configuration
SMARTPROXY_USERNAME = os.getenv('SMARTPROXY_USERNAME')
SMARTPROXY_PASSWORD = os.getenv('SMARTPROXY_PASSWORD')
SMARTPROXY_ENDPOINT = os.getenv('SMARTPROXY_ENDPOINT', 'gate.smartproxy.com:7000')
REQUEST_TIMEOUT = int(os.getenv('REQUEST_TIMEOUT', 10))
MAX_RETRIES = int(os.getenv('MAX_RETRIES', 5))
CONCURRENT_REQUESTS = int(os.getenv('CONCURRENT_REQUESTS', 10))
TARGET_URL = os.getenv('TARGET_URL', 'http://httpbin.org/ip')

# Setup Logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler("proxy_requests.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Define the proxy URL with authentication
proxy_url = f"http://{SMARTPROXY_USERNAME}:{SMARTPROXY_PASSWORD}@{SMARTPROXY_ENDPOINT}"

# Define the proxies dictionary
proxies = {
    "http://": proxy_url,
    "https://": proxy_url,
}

# Define a retry decorator using tenacity
retry_decorator = retry(
    stop=stop_after_attempt(MAX_RETRIES),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type((httpx.RequestError, httpx.HTTPStatusError)),
    reraise=True
)

@retry_decorator
async def fetch(session: httpx.AsyncClient, url: str):
    try:
        response = await session.get(url, timeout=REQUEST_TIMEOUT)
        response.raise_for_status()  # Raise exception for HTTP errors
        data = response.json()
        logger.info(f"Success: {data}")
        return data
    except httpx.HTTPStatusError as exc:
        logger.error(f"HTTP error occurred: {exc.response.status_code} - {exc.response.text}")
        raise
    except httpx.RequestError as exc:
        logger.error(f"Request error occurred: {exc}")
        raise

async def bound_fetch(semaphore: asyncio.Semaphore, session: httpx.AsyncClient, url: str):
    async with semaphore:
        try:
            await fetch(session, url)
        except Exception as e:
            logger.error(f"Failed to fetch {url}: {e}")

async def main():
    # Create a semaphore to limit concurrent requests
    semaphore = asyncio.Semaphore(CONCURRENT_REQUESTS)

    async with httpx.AsyncClient(proxies=proxies) as session:
        tasks = []
        # Example: Sending 50 requests
        for i in range(50):
            task = asyncio.create_task(bound_fetch(semaphore, session, TARGET_URL))
            tasks.append(task)

        # Gather all tasks
        await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(main())
```

### **4. Explanation of the Advanced Components**

#### **a. Asynchronous Requests with `httpx` and `asyncio`**

- **`httpx.AsyncClient`:** Enables making asynchronous HTTP requests, allowing multiple requests to be in-flight concurrently.
- **`asyncio.Semaphore`:** Controls the number of concurrent requests to prevent overwhelming the system or target server.

#### **b. Robust Error Handling & Retries with `tenacity`**

- **`tenacity.retry`:** Decorator that automatically retries a function upon encountering specified exceptions.
- **Exponential Backoff:** Waits progressively longer between retries (e.g., 2s, 4s, 8s) to reduce the load on servers and increase the chances of success.
- **`stop_after_attempt(MAX_RETRIES)`:** Limits the number of retry attempts to prevent infinite loops.

#### **c. Logging**

- **Logging Configuration:** Logs are written both to the console and a file named `proxy_requests.log`.
- **Log Messages:** Provide information about successful requests, HTTP errors, request errors, and failed fetch attempts.

#### **d. Configuration Management with `.env`**

- **Sensitive Data Handling:** Credentials and configuration options are stored securely in a `.env` file.
- **Flexibility:** Easily change configurations without modifying the codebase.

#### **e. Proxy Pool Management**

While Smartproxy's rotating endpoint handles IP rotation automatically, the script includes mechanisms to handle failed proxies gracefully through retries and error handling.

#### **f. Rate Limiting with Semaphore**

The semaphore limits the number of concurrent requests (`CONCURRENT_REQUESTS`) to prevent overwhelming the target server and reduce the likelihood of being blocked.

### **5. Running the Advanced Script**

1. **Ensure `.env` is properly configured** with your Smartproxy credentials and desired settings.
2. **Run the script:**

   ```bash
   python advanced_proxy_request.py
   ```

3. **Monitor Logs:**
   - **Console:** Real-time logs of the script's progress.
   - **Log File (`proxy_requests.log`):** Detailed logs for later analysis.

### **6. Additional Enhancements**

Depending on your specific use case, you might consider the following enhancements:

#### **a. Asynchronous Retry on Specific Status Codes**

Customize retries based on HTTP status codes (e.g., retry on 500 Internal Server Error but not on 404 Not Found).

```python
from tenacity import retry_if_exception_type, retry_if_result

def is_retryable_status(response):
    return response.status_code >= 500

@retry(
    stop=stop_after_attempt(MAX_RETRIES),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=(retry_if_exception_type(httpx.RequestError) | retry_if_result(is_retryable_status)),
    reraise=True
)
async def fetch(session: httpx.AsyncClient, url: str):
    response = await session.get(url, timeout=REQUEST_TIMEOUT)
    if response.status_code >= 500:
        logger.warning(f"Retryable HTTP error: {response.status_code}")
        raise httpx.HTTPStatusError(f"HTTP {response.status_code}", request=response.request, response=response)
    response.raise_for_status()
    data = response.json()
    logger.info(f"Success: {data}")
    return data
```

#### **b. Proxy Validation**

Before sending requests, validate proxies to ensure they're working. This can be done by periodically checking a subset of proxies.

#### **c. Dynamic Target URLs**

Modify the script to accept a list of target URLs, enabling scraping multiple endpoints concurrently.

#### **d. Integration with Scraping Frameworks**

Integrate with frameworks like `Scrapy` for more complex scraping tasks, leveraging Smartproxy within spider middleware.

### **7. Complete Advanced Script Example**

Here's the refined advanced script incorporating all discussed features:

```python
import os
import asyncio
import httpx
import logging
from dotenv import load_dotenv
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

# Load environment variables from .env file
load_dotenv()

# Configuration
SMARTPROXY_USERNAME = os.getenv('SMARTPROXY_USERNAME')
SMARTPROXY_PASSWORD = os.getenv('SMARTPROXY_PASSWORD')
SMARTPROXY_ENDPOINT = os.getenv('SMARTPROXY_ENDPOINT', 'gate.smartproxy.com:7000')
REQUEST_TIMEOUT = int(os.getenv('REQUEST_TIMEOUT', 10))
MAX_RETRIES = int(os.getenv('MAX_RETRIES', 5))
CONCURRENT_REQUESTS = int(os.getenv('CONCURRENT_REQUESTS', 10))
TARGET_URL = os.getenv('TARGET_URL', 'http://httpbin.org/ip')

# Setup Logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler("proxy_requests.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Define the proxy URL with authentication
proxy_url = f"http://{SMARTPROXY_USERNAME}:{SMARTPROXY_PASSWORD}@{SMARTPROXY_ENDPOINT}"

# Define the proxies dictionary
proxies = {
    "http://": proxy_url,
    "https://": proxy_url,
}

# Define a retry decorator using tenacity
retry_decorator = retry(
    stop=stop_after_attempt(MAX_RETRIES),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type((httpx.RequestError, httpx.HTTPStatusError)),
    reraise=True
)

@retry_decorator
async def fetch(session: httpx.AsyncClient, url: str):
    try:
        response = await session.get(url, timeout=REQUEST_TIMEOUT)
        response.raise_for_status()  # Raise exception for HTTP errors
        data = response.json()
        logger.info(f"Success: {data}")
        return data
    except httpx.HTTPStatusError as exc:
        logger.error(f"HTTP error occurred: {exc.response.status_code} - {exc.response.text}")
        raise
    except httpx.RequestError as exc:
        logger.error(f"Request error occurred: {exc}")
        raise

async def bound_fetch(semaphore: asyncio.Semaphore, session: httpx.AsyncClient, url: str, request_id: int):
    async with semaphore:
        try:
            logger.info(f"Request {request_id}: Sending request to {url}")
            await fetch(session, url)
            logger.info(f"Request {request_id}: Completed successfully")
        except Exception as e:
            logger.error(f"Request {request_id}: Failed to fetch {url}: {e}")

async def main():
    # Create a semaphore to limit concurrent requests
    semaphore = asyncio.Semaphore(CONCURRENT_REQUESTS)

    async with httpx.AsyncClient(proxies=proxies) as session:
        tasks = []
        # Example: Sending 50 requests
        for i in range(50):
            task = asyncio.create_task(bound_fetch(semaphore, session, TARGET_URL, i + 1))
            tasks.append(task)

        # Gather all tasks and handle exceptions
        results = await asyncio.gather(*tasks, return_exceptions=True)
        # Optionally, process results here

if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Program interrupted by user.")
    except Exception as e:
        logger.exception(f"An unexpected error occurred: {e}")
```

### **8. Running the Advanced Script**

1. **Ensure All Configurations are Correct:**
   - Your `.env` file should have the correct Smartproxy credentials and desired settings.
2. **Execute the Script:**

   ```bash
   python advanced_proxy_request.py
   ```

3. **Monitor the Output:**
   - **Console:** Real-time updates of request statuses.
   - **Log File (`proxy_requests.log`):** Detailed logs for each request, including successes and failures.

### **9. Conclusion**

This advanced setup provides a robust framework for sending HTTP requests through Smartproxy's rotating proxies using Python. It leverages asynchronous programming for efficiency, implements comprehensive error handling and retries, and maintains detailed logs for monitoring and debugging.

#### **Next Steps:**

- **Customization:** Adapt the script to fit your specific use case, such as scraping multiple URLs or integrating with data pipelines.
- **Scalability:** Adjust `CONCURRENT_REQUESTS` based on your system's capabilities and the target server's tolerance.
- **Security:** Ensure your Smartproxy credentials are securely stored and managed.
- **Optimization:** Fine-tune retry strategies and backoff parameters based on observed performance and failure rates.

Feel free to reach out if you need further assistance or have specific requirements to address!
