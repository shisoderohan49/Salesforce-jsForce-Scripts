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
