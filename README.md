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
    console.log('-------------------------------');
    console.log(`Test Result for ${testResult.testClassName}`);
    console.log('-------------');
    console.log(`${testResult.testClassName} Successful Test Methods : `);
    console.log(testResult.successes);
    console.log('-------------');
    console.log(`${testResult.testClassName} Failure Test Methods : `);
    console.log(testResult.failures);
    console.log('-------------');
    console.log(`${testResult.testClassName} Number of Failures : `);
    console.log(testResult.numFailures);
    console.log('-------------');

    if(testResult.codeCoverage.length !== 0){
        console.log(`${testResult.testClassName} Code Coverage :\n`);
        testResult.codeCoverage.forEach(coverage => {
            console.log(`Class Name : ${coverage.className}`);
            console.log(`Code Coverage Percentage : ${coverage.codeCoveragePercentage}\n`);
        });
    }

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

    console.log('------------TEST RESULT FAILURES -------------');
    testResultFailures.forEach(testResultFailure => {
        console.log(`Test Class Name : ${testResultFailure.testClassName}`);
        console.log('-------------');
        console.log('Failures : ');
        console.log(testResultFailure.failures);
        console.log('-------------');
        console.log('Low Code Coverage Classes : ');
        console.log(testResultFailure.lowCodeCoverage);
        console.log('-------------');
    });
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
