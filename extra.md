```
(.venv) PS C:\Users\attic\gitrepos\Codepath\AI-201\week-4\provenance-guard> 1..12 | ForEach-Object {
>>   try {
>>     $r = Invoke-WebRequest -Method POST http://localhost:5000/submit -ContentType "application/json" -Body '{"text": "Rate limit test.", "creator_id": "ratelimit-test"}'
>>     Write-Host $r.StatusCode
>>   } catch {
>>     Write-Host $_.Exception.Response.StatusCode.value__
>>   }
>> }
200
200
200
200
200
200
200
200
200
200
429
429
```
