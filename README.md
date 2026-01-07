class Logger {
    constructor(uiElementId = 'log-output') {
        this.logElement = document.getElementById(uiElementId) || console;
        this.isConsole = (this.logElement === console);
    }

    _timestamp() {
        return new Date().toLocaleTimeString('en-US', { hour12: false });
    }

    log(message, type = 'INFO') {
        const fullMessage = `[${this._timestamp()}] [${type}] ${message}`;
        if (this.isConsole) {
            console.log(fullMessage);
        } else {
            const line = document.createElement('div');
            line.className = `log-line log-${type.toLowerCase()}`;
            line.textContent = fullMessage;
            this.logElement.prepend(line);
            if (this.logElement.children.length > 50) {
                this.logElement.lastChild.remove();
            }
        }
    }

    error(message) {
        this.log(message, 'ERROR');
    }

    success(message) {
        this.log(message, 'SUCCESS');
    }

    warn(message) {
        this.log(message, 'WARN');
    }
}

class GitHubClient {
    constructor(token, owner, repo, logger) {
        this.token = token;
        this.owner = owner;
        this.repo = repo;
        this.baseURL = `https://api.github.com/repos/${owner}/${repo}`;
        this.logger = logger;

        if (!this.token) {
            this.logger.error("GitHub token is required.");
            throw new Error("GitHub token not set.");
        }
    }

    _fetch(endpoint, options = {}) {
        const headers = {
            'Authorization': `token ${this.token}`,
            'Accept': 'application/vnd.github.v3+json',
            'Content-Type': 'application/json',
            ...options.headers
        };

        return fetch(`${this.baseURL}${endpoint}`, {
            ...options,
            headers: headers
        }).then(res => {
            if (!res.ok) {
                throw new Error(`GitHub API Error (${res.status}): ${res.url}`);
            }
            return res.json();
        });
    }

    async getReference(ref = 'heads/main') {
        this.logger.log(`Fetching reference for ${ref}...`);
        return this._fetch(`/git/ref/${ref}`);
    }

    async createBranch(newBranchName, sha) {
        this.logger.warn(`Creating new branch: ${newBranchName}`);
        return this._fetch('/git/refs', {
            method: 'POST',
            body: JSON.stringify({
                ref: `refs/heads/${newBranchName}`,
                sha: sha
            })
        });
    }

    async getTree(sha, recursive = false) {
        this.logger.log(`Fetching tree structure for SHA: ${sha.substring(0, 7)}`);
        return this._fetch(`/git/trees/${sha}${recursive ? '?recursive=1' : ''}`);
    }

    async getFileContent(filePath, branch) {
        this.logger.log(`Reading file: ${filePath}`);
        const data = await this._fetch(`/contents/${filePath}?ref=${branch}`);
        
        if (data.encoding !== 'base64') {
             // Handle raw text/other encodings if necessary, but GitHub defaults to base64
             return atob(data.content);
        }

        return atob(data.content);
    }
    
    async commitChanges(branchName, commitMessage, filesToUpdate) {
        this.logger.log(`Preparing commit for ${filesToUpdate.length} files...`);
        
        const baseRef = await this.getReference(`heads/${branchName}`);
        const baseCommitSha = baseRef.object.sha;
        
        // 1. Get the current tree
        const baseCommit = await this._fetch(`/git/commits/${baseCommitSha}`);
        const baseTreeSha = baseCommit.tree.sha;

        // 2. Create new blobs and tree entries
        const newTreePromises = filesToUpdate.map(async file => {
            const contentBase64 = btoa(unescape(encodeURIComponent(file.newContent)));
            
            // Create Blob
            const blob = await this._fetch('/git/blobs', {
                method: 'POST',
                body: JSON.stringify({
                    content: contentBase64,
                    encoding: 'base64'
                })
            });

            return {
                path: file.path,
                mode: '100644', // File mode
                type: 'blob',
                sha: blob.sha
            };
        });

        const treeEntries = await Promise.all(newTreePromises);
        
        // 3. Create the new Tree
        const newTree = await this._fetch('/git/trees', {
            method: 'POST',
            body: JSON.stringify({
                base_tree: baseTreeSha,
                tree: treeEntries
            })
        });

        // 4. Create the new Commit
        const newCommit = await this._fetch('/git/commits', {
            method: 'POST',
            body: JSON.stringify({
                message: commitMessage,
                tree: newTree.sha,
                parents: [baseCommitSha]
            })
        });

        // 5. Update the Branch Reference
        await this._fetch(`/git/refs/heads/${branchName}`, {
            method: 'PATCH',
            body: JSON.stringify({
                sha: newCommit.sha
            })
        });

        this.logger.success(`Successfully pushed commit: ${newCommit.sha.substring(0, 7)}`);
        return newCommit;
    }
}

class GeminiClient {
    constructor(apiKey, model = 'gemini-2.5-pro', logger) {
        this.apiKey = apiKey;
        this.model = model;
        this.logger = logger;
        this.baseURL = `https://generativelanguage.googleapis.com/v1beta/models/${this.model}:generateContent?key=${this.apiKey}`;
        
        if (!this.apiKey) {
            this.logger.error("Gemini API key is required.");
            throw new Error("Gemini API key not set.");
        }
    }

    async analyzeAndGenerateCode(codebaseStructure) {
        this.logger.log("Sending codebase for AI analysis and refinement...");

        const systemInstruction = `You are the Dalek Linear Evolution System (v18). Your task is to analyze the provided JavaScript/HTML codebase structure, identify the single most critical or beneficial optimization/fix, and generate the *complete, improved contents* for *only* the files you intend to modify. 
        Respond ONLY with a JSON array formatted exactly as follows: 
        [{ "path": "path/to/file.js", "newContent": "The full, optimized content of the file..." }]
        DO NOT include any files you are not modifying. Ensure the JSON is valid and complete.`;

        const prompt = `Codebase Structure and Contents:\n\n${JSON.stringify(codebaseStructure, null, 2)}`;
        
        const response = await fetch(this.baseURL, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                contents: [{ role: "user", parts: [{ text: prompt }] }],
                config: {
                    systemInstruction: systemInstruction,
                    temperature: 0.1, // Ensure stable, reliable code output
                    responseMimeType: "application/json"
                }
            })
        });

        if (!response.ok) {
            const errorText = await response.text();
            throw new Error(`Gemini API Error (${response.status}): ${errorText}`);
        }

        const jsonResponse = await response.json();
        
        // The API returns the response wrapped in text
        const generatedText = jsonResponse.candidates[0].content.parts[0].text.trim();
        
        try {
            const updates = JSON.parse(generatedText);
            this.logger.success(`AI returned ${updates.length} files for update.`);
            return updates;
        } catch (e) {
            this.logger.error("Failed to parse AI output as JSON.");
            throw new Error(`Invalid JSON structure received from AI: ${generatedText.substring(0, 200)}`);
        }
    }
}

class EvolutionEngine {
    constructor(config) {
        this.config = {
            OWNER: 'craighckby-stack',
            REPO: 'ai-scaffold',
            SOURCE_BRANCH: 'main',
            ...config
        };
        this.logger = new Logger(this.config.LOG_ELEMENT_ID || 'log-output');
        this.github = null;
        this.gemini = null;
    }

    async initialize() {
        this.logger.log("Initializing Dalek Linear Evolution System v18...");
        
        const githubToken = this.config.GITHUB_TOKEN || prompt("Please enter your GitHub Personal Access Token:");
        if (!githubToken) throw new Error("Initialization failed: GitHub Token missing.");
        
        const geminiKey = this.config.GEMINI_KEY || prompt("Please enter your Gemini API Key:");
        if (!geminiKey) throw new Error("Initialization failed: Gemini Key missing.");
        
        this.github = new GitHubClient(githubToken, this.config.OWNER, this.config.REPO, this.logger);
        this.gemini = new GeminiClient(geminiKey, this.config.MODEL, this.logger);

        // Optional Firebase setup placeholder
        if (typeof firebase !== 'undefined' && this.config.FIREBASE_CONFIG) {
            try {
                 firebase.initializeApp(this.config.FIREBASE_CONFIG);
                 this.db = firebase.firestore();
                 this.logger.log("Firebase persistence enabled.", 'SETUP');
            } catch(e) {
                 this.logger.warn("Firebase initialization failed. Continuing without persistence.");
            }
        }

        this.logger.success("System initialized successfully.");
    }

    async runEvolutionCycle(cycle = 1, totalCycles = 5) {
        this.logger.log(`--- Starting Evolution Cycle ${cycle}/${totalCycles} ---`, 'CYCLE');

        const evolutionBranch = `dalek-evolution-${Date.now()}`;
        let baseRef, baseSha;
        let fileContents = {};

        try {
            // 1. Timeline Establishment (Create Branch)
            baseRef = await this.github.getReference(`heads/${this.config.SOURCE_BRANCH}`);
            baseSha = baseRef.object.sha;
            await this.github.createBranch(evolutionBranch, baseSha);
            this.logger.success(`Evolution established on branch: ${evolutionBranch}`);

            // 2. Code Scanning (Read codebase)
            const treeData = await this.github.getTree(baseSha, true);
            const filesToAnalyze = treeData.tree
                .filter(item => item.type === 'blob' && !item.path.startsWith('.github') && !item.path.startsWith('node_modules'))
                .map(item => item.path);

            this.logger.log(`Found ${filesToAnalyze.length} files for analysis.`);

            // Fetch contents in parallel
            const fetchPromises = filesToAnalyze.map(async (path) => {
                const content = await this.github.getFileContent(path, evolutionBranch);
                fileContents[path] = content;
            });
            await Promise.all(fetchPromises);

            // 3. AI Analysis & Refinement
            const updates = await this.gemini.analyzeAndGenerateCode(fileContents);
            
            if (updates.length === 0) {
                 this.logger.warn("AI returned no modifications. Ending evolution early.");
                 return;
            }

            // 4. Commit Changes
            const commitMessage = `# Dalek Linear Evolution - Recursive Commits v${this.config.VERSION || 18} (Cycle ${cycle})\nAutomated refinement targeting code stability and optimization.`;

            await this.github.commitChanges(evolutionBranch, commitMessage, updates);

            // 5. Repeat / Recursion
            this.logger.success(`Cycle ${cycle} complete. Changes committed to ${evolutionBranch}.`);
            
            // Log persistence (Optional)
            if (this.db) {
                await this.db.collection("evolution_history").add({
                    cycle,
                    timestamp: firebase.firestore.FieldValue.serverTimestamp(),
                    branch: evolutionBranch,
                    changes: updates.map(u => u.path)
                });
            }


            if (cycle < totalCycles) {
                // Set the current evolution branch as the source for the next cycle
                this.config.SOURCE_BRANCH = evolutionBranch; 
                await this.runEvolutionCycle(cycle + 1, totalCycles);
            } else {
                this.logger.success(`--- Evolution Complete: ${totalCycles} Cycles Finalized. ---`, 'COMPLETE');
                this.logger.log(`The final evolved branch is: ${evolutionBranch}`);
            }

        } catch (error) {
            this.logger.error(`Evolution failed during Cycle ${cycle}: ${error.message}`);
            this.logger.error("Please check configurations and API keys.");
        }
    }
}

// Example Initialization (Requires external configuration via index.html or global vars)
/*
async function startEvolution() {
    const config = {
        GITHUB_TOKEN: "YOUR_TOKEN_HERE",
        GEMINI_KEY: "YOUR_KEY_HERE",
        OWNER: 'craighckby-stack',
        REPO: 'ai-scaffold',
        VERSION: 18,
        LOG_ELEMENT_ID: 'evolution-log-output',
        // FIREBASE_CONFIG: { ... } // Uncomment and configure if using persistence
    };
    
    const engine = new EvolutionEngine(config);
    try {
        await engine.initialize();
        // Start with the configured SOURCE_BRANCH (e.g., 'main')
        engine.runEvolutionCycle(1, 5); 
    } catch (e) {
        document.getElementById('evolution-log-output').innerHTML += `<div class="log-error">SETUP FAILURE: ${e.message}</div>`;
    }
}
*/
// startEvolution();
