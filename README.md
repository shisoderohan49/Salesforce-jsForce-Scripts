# Salesforce jsforce Scripts
## Pre-requisites
### Installiing jsForce  
`npm install jsforce`  

### Establising Salesforce Connection 
I mainly use the Session Id to establish connection for the jsforce Connection Object.  
- Retrieving Session Id by running this code in Salesforce Developer Console's `Execute Anonymous Apex`  
`System.debug('Session id '+UserInfo.getOrganizationId() + UserInfo.getSessionId().substring(15));`
  
- Establising connection in script using Session Id
```
async function getConnectionObj(instanceUrl,sessionId){
    const conn = new jsforce.Connection({
        instanceUrl: instanceUrl,
        serverUrl: instanceUrl,
        sessionId: sessionId
    })
    return conn;
}
```

# Scripts 

<details>
  <summary>List of Scripts</summary>

  - [Running Test Classes and retrieving respective Code Coverage for particular apex classes](#running-test-classes-and-retrieving-respective-code-coverage-for-particular-apex-classes)
  - [Inserting a field in multiple List Views of a certain Object at a certain position](#inserting-a-field-in-multiple-list-views-of-a-certain-object-at-a-certain-position)
  - [Update Default External Access of list of Objects using jsForce](#update-default-external-access-of-list-of-objects-using-jsforce)
</details>

## Running Test Classes and retrieving respective Code Coverage for particular apex classes
[Back to main](#scripts)

```
var jsforce = require('jsforce');

async function getConnectionObj(instanceUrl,sessionId){
    const conn = new jsforce.Connection({
        instanceUrl: instanceUrl,
        serverUrl: instanceUrl,
        sessionId: sessionId
    });
    return conn;
}

function printTestResult(testResult){
    console.log(`Test Result for ${testResult.testClassName}`);
    console.table([
        { 'Successful Test Methods': testResult.successes.join(', ') }
    ])

    console.table([
        { 
            'Failure Test Methods': testResult.failures.join(', '),
            'Number of Failures': testResult.numFailures 
        }
    ])

    console.log(`Code Coverage for ${testResult.testClassName}`);
    console.table(testResult.codeCoverage.map(coverage => {
        return {
            'Class Name': coverage.className,
            'Code Coverage Percentage': coverage.codeCoveragePercentage
        }
    }));
    console.log(`Test Result for ${testResult.testClassName} Complete\n`);
}

async function main(testMap,instanceUrl,sessionId){
    const conn = await getConnectionObj(instanceUrl,sessionId);

    var testClasses = Array.from(testMap.keys());
    var testNodes = testClasses.map(testClass => {
        return { className: testClass };
    })
    var testResultFailures = [];

    for(var index = 0; index < testNodes.length; index++){
        var testNode = testNodes[index];
        var codeCoverageClasses = testMap.get(testNode.className);

        var testResults = await conn.tooling.runTestsSynchronous({
            tests: [testNode]
        });

        var testResult = {};
        testResult.testClassName = testNode.className;
        testResult.codeCoverage = testResults.codeCoverage
                                  .filter(classCoverage => codeCoverageClasses.includes(classCoverage.name))
                                  .map(coverage => {
                                        let codeCoveragePercentage = ((coverage.numLocations - coverage.numLocationsNotCovered)/coverage.numLocations) * 100;
                                        return {
                                            className: coverage.name,
                                            codeCoveragePercentage: codeCoveragePercentage
                                        }
                                  });

        testResult.successes = testResults.successes.map(success => success.methodName);
        testResult.failures = testResults.failures.map(failure => {
            return {
                methodName: failure.methodName,
                message: failure.message,
                stackTrace: failure.stackTrace
            }
        });
        testResult.numFailures = testResults.numFailures;

        printTestResult(testResult);

        var testResultFailure = {};
        testResultFailure.testClassName = testResult.testClassName;
        testResultFailure.failures = testResult.failures;
        testResultFailure.lowCodeCoverage = testResult.codeCoverage.filter(coverage => coverage.codeCoveragePercentage < 75);

        testResultFailures.push(testResultFailure);
    }

    console.log('LIST OF TEST RESULT FAILURES')
    console.table(testResultFailures.map(testResultFailure => {
        return {
            'Test Class Name': testResultFailure.testClassName,
            'Failures': testResultFailure.failures.join(', '),
            'Low Code Coverage Classes': testResultFailure.lowCodeCoverage.join(', ')
        }
    }))
}

```
Running the script 
```
var testMap = new Map();
testMap.set('TESTCLASSNAME1',[]);
testMap.set('TESTCLASSNAME2',['APEXCLASS1','APEXCLASS2']);

var instanceURL = '<YOUR_DOMAIN_URL>';
var sessionId = '<YOUR_SESSION_ID>';

main(testMap,instanceURL,sessionId);
```

## Inserting a field in multiple List Views of a certain Object at a certain position
[Back to main](#scripts)

We utilize Metadata API via jsforce to implement this requirement.
The following steps are implemented 

- Get Developer Names of required List Views of the object
- Create equivalent List View Metadata API names from this list
- Divide the returned array/list into smaller chunks of same size to avoid any Batch Size error for Salesforce Metadata API Call Limits.
- Retrieve the Metadata of the given chunk
- Check whether the field is already present in the Columns of the retrieved metadata of List View or not
- If Not, add the field to the specified position
- Update the metadata of all List Views in the chunk
- Repeat this process for all the chunks

We create a function to convert a array into a array of arrays of the same length  
```
function returnChunkArr(arr,chunkSize){
    var result = [];
    var index = 0;
    while(index < arr.length){
        result.push(arr.slice(index,index + chunkSize));
        index += chunkSize;
    }
    return result;
}
```

We have the `getConnectionObj` function to get the required connection object to connect to our instance.
```
var jsforce = require('jsforce');

async function getConnectionObj(instanceUrl,sessionId){
    const conn = new jsforce.Connection({
        instanceUrl: instanceUrl,
        serverUrl: instanceUrl,
        sessionId: sessionId
    })
    return conn;
}
```

The main function implements the logic to update the List Views  
We read the metadata of array of list views via `conn.metadata.read('ListView',listViewArr)`  
```
async function main(sObjectApiName,fieldApiName,fieldPosition,instanceUrl,sessionId){
    //get the conn object via getConnectionObj
    var conn = await getConnectionObj(instanceUrl, sessionId);

    //Creating Query for retrieving DeveloperName of List Views for a certain object
    var listViewQuery = `SELECT Id,DeveloperName FROM ListView WHERE SobjectType = '${sObjectApiName}'`;
    var listViews = await conn.query(listViewQuery);

    //Converting DeveloperName to Metadata API Name , FIELD_API_NAME => SOBJECT_API_NAME.FIELD_API_NAME
    var viewDeveloperNames = listViews.records.map(view => `${sObjectApiName}.${view.DeveloperName}`);

    //using our created returnChunkArr function to convert the viewDeveloperNames array to array of arrays of equal size - 5 for example
    var arrListViews = returnChunkArr(viewDeveloperNames, 5);

    //reading metadata of list views
    var listViewMetadata = await conn.metadata.read('ListView', arrListViews[0]);
    console.dir(listViewMetadata, { depth: null });
}
```
The retrieved metadata of list view is of this format
```
{
    fullName: 'SOBJECT_API_NAME.LIST_VIEW_API_NAME',
    columns: [
      'FIELD_API_NAME_1',
      'FIELD_API_NAME_2',
      'FIELD_API_NAME_3',
      'SOBJECT_API_NAME.STANDARD_FIELD_API_NAME'
    ],
    filterScope: 'Everything',
    filters: [
      { field: 'FIELD_API_NAME_2', operation: 'equals', value: 'SAMPLE_VALUE' },
      { field: 'FIELD_API_NAME_3', operation: 'equals', value: 'SAMPLE_VALUE' }
    ],
    label: 'LIST_VIEW_LABEL'
}
```
Since we have to insert a field into columns of a list view, we have to update the columns property.  
(standard fields may have to be inserted in the format - `SOBJECT_API_NAME.STANDARD_FIELD_API_NAME`, not completely sure about this, will need to be checked later)  
Let's implement this logic in another function `workOnChunk`, we'll also have to give the connection object as a argument.  
We update the list view metadata by using the function `conn.metadata.update('ListView',listViewMetadata)`  
```
async function workOnChunk(conn,chunk,fieldApiName,fieldPosition){
    const listViewMetadata = await conn.metadata.read('ListView', chunk);
    
    for(var index = 0; index < listViewMetadata.length; index++){
        var meta = listViewMetadata[index];
        if(!meta.columns.includes(fieldApiName)){
            if(meta.columns.length < fieldPosition){
                meta.columns.push(fieldApiName);
            }else{
                meta.columns.splice(fieldPosition - 1,0,fieldApiName);
            }
        }
    }

    var results = await conn.metadata.update('ListView',listViewMetadata);
    return results;
}
```
The results show whether the list view update was successful or not and is of this format
```
{
      "fullName": "SOBJECT_API_NAME.LIST_VIEW_API_NAME",
      "success": true,
      "errors": []
}
```
In case of error, `success` and `error` key values will be different.  

This will be the finalized script. 

```
var jsforce = require('jsforce');
var fs = require('fs');

async function getConnectionObj(instanceUrl,sessionId){
    const conn = new jsforce.Connection({
        instanceUrl: instanceUrl,
        serverUrl: instanceUrl,
        sessionId: sessionId
    })
    return conn;
}

function returnChunkArr(arr,chunkSize){
    var result = [];
    var index = 0;
    while(index < arr.length){
        result.push(arr.slice(index,index + chunkSize));
        index += chunkSize;
    }
    return result;
}

async function workOnChunk(conn,chunk,fieldApiName,fieldPosition){
    const listViewMetadata = await conn.metadata.read('ListView', chunk);
    
    for(var index = 0; index < listViewMetadata.length; index++){
        var meta = listViewMetadata[index];
        if(!meta.columns.includes(fieldApiName)){
            if(meta.columns.length <= fieldPosition){
                meta.columns.push(fieldApiName);
            }else{
                meta.columns.splice(fieldPosition - 1,0,fieldApiName);
            }
        }
    }

    var results = await conn.metadata.update('ListView',listViewMetadata);
    return results;
}

async function main(sObjectApiName,fieldApiName,fieldPosition,instanceUrl,sessionId){
    var conn = await getConnectionObj(instanceUrl, sessionId);

    var listViewQuery = `SELECT Id,DeveloperName FROM ListView WHERE SobjectType = '${sObjectApiName}' LIMIT 10`;
    var listViews = await conn.query(listViewQuery);

    var viewDeveloperNames = listViews.records.map(view => `${sObjectApiName}.${view.DeveloperName}`);

    var arrListViews = returnChunkArr(viewDeveloperNames, 10);
    var results = [];

    for(var index = 0; index < arrListViews.length; index++){
        var chunk = arrListViews[index];
        results.push(await workOnChunk(conn,chunk,fieldApiName,fieldPosition));
    }

    fs.writeFileSync('list_view_update_results.json', JSON.stringify(results, null, 2));
}

var sobjectApiName = 'INPUT_SOBJECT_API_NAME';
var fieldApiName = 'INPUT_FIELD_API_NAME';
var fieldPosition = INPUT_FIELD_POSITION; // 1-based index
var sessionId = 'INPUT_SESSION_ID';
var instanceUrl = 'INPUT_INSTANCE_URL';
main(sobjectApiName,fieldApiName,fieldPosition,instanceUrl,sessionId);
```
  
Change the ListView Query or provide the list view developer names yourself as per your requirement.  

## Update Default External Access of list of Objects using jsForce
[Back to main](#scripts)

```
var jsforce = require('jsforce');

async function getConnectionObj(instanceUrl,sessionId){
    const conn = new jsforce.Connection({
        instanceUrl: instanceUrl,
        serverUrl: instanceUrl,
        sessionId: sessionId
    })
    return conn;
}

async function main(instanceUrl,sessionId,customObjects){
    var conn = await getConnectionObj(instanceUrl,sessionId);

    var metadataChunks = returnChunkArr(customObjects,10);
    
    for(let index = 0;index < metadataChunks.length; index++){
        var chunk = metadataChunks[index];
        var metadataResult = await conn.metadata.read('CustomObject',chunk);
        metadataResult.forEach(metadataObject => {
            metadataObject.externalSharingModel = 'Private';
        })

        var updateResult = await conn.metadata.update('CustomObject',metadataResult);
        console.dir(updateResult,{depth:null});
    }
}

function returnChunkArr(zipcodeArr,chunkSize){
    var result = [];
    var i = 0;
    while(i < zipcodeArr.length){
        result.push(zipcodeArr.slice(i,i + chunkSize));
        i += chunkSize;
    }
    return result;
}


var instanceUrl = 'INSTANCE_URL';
var sessionId = 'SESSION_ID';
var customObjects = [/*LIST OF CUSTOM OBJECTS*/];
main(instanceUrl,sessionId,customObjects);
```
