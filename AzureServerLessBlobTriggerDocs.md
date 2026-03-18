# 1. Create a Storage

# 2. Create an event grid trigger blob events
- Open VSCode Press F1 choose "create function"
- choose the runtime
- choose the subscription
- choose the function app

# 3. Clean up before deployment
```bash
rm -rf __pycache__
rm -rf .python_packages
find . -type f -name "*.pyc" -delete
func azure functionapp publish sandeepogfunctionapp --build remote --force
```
# 4. Sample EventGridTrigger Python Code
```python
import azure.functions as func
from azure.storage.blob import BlobServiceClient
import logging
app = func.FunctionApp()
@app.event_grid_trigger(arg_name="azeventgrid")
def eventgridtrigger(azeventgrid: func.EventGridEvent):
    event_data = azeventgrid.get_json()
    
    # 2. Log the event details
    logging.info(f"Event Type: {azeventgrid.event_type}")
    logging.info(f"Subject (Blob Path): {azeventgrid.subject}")
    
    # 3. Extract the Blob URL from the payload
    # For BlobCreated events, the URL is inside the 'url' key of the data object
    blob_url = event_data.get('url','null')
    logging.info(f"New Blob detected at: {blob_url}")
```

# 5. Create the Event Subscription in Storage Account
- Azure Portal -> Storage Account -> <YOUR_STORAGE_ACCOUNT> -> Events.
- Provide subscription name
- Provide topic name
- Choose the action when the event should trigger. (check all options)
- choose endpoint type as Azure function
- choose the event grid endpoint.
- Note: only event grid triggers are listed in the endpoints of azure function in event subscription creation.

# 6. Test the trigger
- Upload the file to the container 
- check the logs 
