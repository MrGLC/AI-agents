You are helping test a machine learning API endpoint.

Follow these steps:

1. **Get API details**:
   - Ask for the API URL/endpoint
   - Request example input format
   - Clarify authentication requirements (API key, JWT, etc.)
   - Understand expected response format

2. **Create comprehensive tests**:
   - **Valid input test**: Test with correct, expected input
   - **Invalid input tests**: Missing fields, wrong types, out-of-range values
   - **Edge cases**: Empty values, zeros, very large/small numbers, boundary values
   - **Error handling**: Verify appropriate error messages
   - **Performance**: Measure response time

3. **Generate test code**:
   Create tests using appropriate tools:
   - Python: `requests` library with `pytest`
   - curl commands for manual testing
   - Postman/Insomnia collections if preferred

4. **Test scenarios to cover**:
   ```python
   # Valid prediction
   # Missing required fields
   # Wrong data types
   # Out of range values
   # Null/None values
   # Very large inputs
   # Batch predictions (if supported)
   # Concurrent requests
   # Response time benchmarks
   ```

5. **Run tests and report results**:
   - Execute the tests
   - Document any failures or issues
   - Measure performance metrics
   - Provide recommendations for fixes

6. **Create test documentation**:
   - Example requests and responses
   - Common error codes and meanings
   - Performance benchmarks

Ask for API details and proceed with testing.
