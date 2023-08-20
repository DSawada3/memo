# memo

"explicitFolding.rules": [
    



        
    ],
    "[cpp]": {
        "explicitFolding.rules": [
            {
                "beginRegex": "#if(?:n?def)?",
                "middleRegex": "#el(?:se|if)",
                "endRegex": "#endif",
                "autoFold": true
            },
            {
                "begin": "//",
                "continuationRegex": "coverity.*",
                "autoFold": true
            },
            {
                "begin": "/*",
                "end": "*/",
                "nested": false,
                "autoFold": true
            },
        ]
    },