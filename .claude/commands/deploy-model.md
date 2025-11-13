You are helping deploy a machine learning model to production.

Follow these steps:

1. **Identify the model**:
   - Ask which model file needs to be deployed
   - Check model format (PyTorch .pth, TensorFlow SavedModel, Keras .h5, pickle .pkl, etc.)
   - Verify model file exists and can be loaded

2. **Determine deployment target**:
   - Ask user: API endpoint (Flask/FastAPI), serverless (Lambda/Cloud Functions), TensorFlow.js (browser), mobile (TFLite), or container (Docker)?
   - Clarify infrastructure preferences (AWS, GCP, Azure, Heroku, local)

3. **Create deployment artifacts**:
   - Generate API code (Flask/FastAPI endpoint)
   - Create Dockerfile if containerizing
   - Generate requirements.txt with dependencies
   - Add health check endpoints
   - Include model loading and inference code

4. **Add error handling and validation**:
   - Input validation
   - Error handling with meaningful messages
   - Logging configuration
   - Request timeouts

5. **Document the deployment**:
   - API endpoint documentation
   - Input/output schema
   - Example requests
   - Environment variables needed

6. **Test the deployment**:
   - Provide test commands
   - Sample curl requests
   - Verification steps

Ask clarifying questions if needed before proceeding.
