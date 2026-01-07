class Logger {
    constructor(id = 'log-output') {
        this.el = document.getElementById(id) || console;
        this.isConsole = (this.el === console);
    }

    log(message, type = 'INFO') {
        const ts = new Date().toLocaleTimeString('en-US', { hour12: false });
        const msg = `[${ts}] [${type}] ${message}`;
        
        if (this.isConsole) {
            console.log(msg);
            return;
        }

        const line = document.createElement('div');
        line.className = `log-line log-${type.toLowerCase()}`;
        line.textContent = msg;
        this.el.prepend(line);
        
        while (this.el.children.length > 50) {
            this.el.lastChild.remove();
        }
    }

    error(m) { this.log(m, 'ERROR'); }
    success(m) { this.log(m, 'SUCCESS'); }
    warn(m) { this.log(m, 'WARN'); }
}

class GitHubClient {
    constructor(token, owner, repo, logger) {
        this.logger = logger;
        this.baseURL = `https://api.github.com/repos/${owner}/${repo}`;
        this.defaultHeaders = {
            'Authorization': `token ${token}`,
            'Accept': 'application/vnd.github.v3+json',
            'Content-Type': 'application/json',
        };

        if (!token) {
            logger.error("GitHub token is required.");
            throw new Error("GitHub token not set.");
        }
    }

    async _fetch(endpoint, options = {}) {
        const mergedHeaders = { ...this.defaultHeaders, ...options.headers };
        
        const response = await fetch(`${this.baseURL}${endpoint}`, {
            ...options,
            headers: mergedHeaders
        });
        
        if (!response.ok) {
            const errorBody = await response.json().catch(() => ({}));
            const message = errorBody.message || `GitHub API Error (${response.status})`;
            throw new Error(`${message}: ${response.url}`);
        }
        
        if (response.status === 204) return null;
        
        return response.json();
    }

    getReference(ref = 'heads/main') {
        this.logger.log(`Fetching ref ${ref}...`);
        return this._fetch(`/git/ref/${ref}`);
    }

    createBranch(newBranchName, sha) {
        this.logger.warn(`Creating branch: ${newBranchName}`);
        return this._fetch('/git/refs', {
            method: 'POST',
            body: JSON.stringify({ ref: `refs/heads/${newBranchName}`, sha })
        });
    }

    getTree(sha, recursive = false) {
        this.logger.log(`Fetching tree for SHA: ${sha.substring(0, 7)}`);
        return this._fetch(`/git/trees/${sha}${recursive ? '?recursive=1' : ''}`);
    }

    async getFileContent(filePath, branch) {
        this.logger.log(`Reading file: ${filePath}`);
        const data = await this._fetch(`/contents/${filePath}?ref=${branch}`);
        
        // Remove whitespace and decode base64
        return atob(data.content.replace(/\s/g, ''));
    }
    
    async commitChanges(branchName, commitMessage, filesToUpdate) {
        this.logger.log(`Preparing commit for ${filesToUpdate.length} files...`);
        
        const baseRef = await this.getReference(`heads/${branchName}`);
        const baseCommitSha = baseRef.object.sha;
        const baseCommit = await this._fetch(`/git/commits/${baseCommitSha}`);
        const baseTreeSha = baseCommit.tree.sha;

        const toBase64 = (str) => btoa(unescape(encodeURIComponent(str)));

        const newTreeEntries = await Promise.all(filesToUpdate.map(async file => {
            const contentBase64 = toBase64(file.newContent);
            
            const blob = await this._fetch('/git/blobs', {
                method: 'POST',
                body: JSON.stringify({ content: contentBase64, encoding: 'base64' })
            });

            return {
                path: file.path,
                mode: '100644', 
                type: 'blob',
                sha: blob.sha
            };
        }));
        
        const newTree = await this._fetch('/git/trees', {
            method: 'POST',
            body: JSON.stringify({ base_tree: baseTreeSha, tree: newTreeEntries })
        });

        const newCommit = await this._fetch('/git/commits', {
            method: 'POST',
            body: JSON.stringify({
                message: commitMessage,
                tree: newTree.sha,
                parents: [baseCommitSha]
            })
        });

        await this._fetch(`/git/refs/heads/${branchName}`, {
            method: 'PATCH',
            body: JSON.stringify({ sha: newCommit.sha, force: false })
        });

        this.logger.success(`Pushed commit: ${newCommit.sha.substring(0, 7)}`);
        return newCommit;
    }
}

class GeminiClient {
    constructor(apiKey, model = 'gemini-2.5-pro', logger) {
        this.logger = logger;
        if (!apiKey) {
            logger.error("Gemini API key is required.");
            throw new Error("Gemini API key not set.");
        }
        this.baseURL = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`;
    }

    async analyzeAndGenerateCode(codebaseStructure) {
        this.logger.log("Sending codebase for AI analysis...");

        const systemInstruction = `You are the Dalek Linear Evolution System (v18). Your task is to analyze the provided JavaScript/HTML codebase structure, identify the single most critical or beneficial optimization/fix, and generate the *complete, improved contents* for *only* the files you intend to modify. Respond ONLY with a JSON array formatted exactly as follows: [{ "path": "path/to/file.js", "newContent": "The full, optimized content of the file..." }] DO NOT include any files you are not modifying. Ensure the JSON is valid and complete.`;
        
        const prompt = JSON.stringify(codebaseStructure);
        
        const payload = {
            contents: [{ role: "user", parts: [{ text: `Codebase Structure and Contents:\n${prompt}` }] }],
            config: {
                systemInstruction: systemInstruction,
                temperature: 0.1, 
                responseMimeType: "application/json"
            }
        };

        const response = await fetch(this.baseURL, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        if (!response.ok) {
            const errorText = await response.text();
            throw new Error(`Gemini API Error (${response.status}): ${errorText}`);
        }

        const jsonResponse = await response.json();
        const candidate = jsonResponse.candidates?.[0];

        if (!candidate || !candidate.content || candidate.content.parts.length === 0) {
             throw new Error("Invalid response structure received from AI.");
        }
        
        const generatedText = candidate.content.parts[0].text.trim();
        
        try {
            const updates = JSON.parse(generatedText);
            
            if (!Array.isArray(updates)) {
                 throw new Error("AI output was parsed but is not a root array.");
            }

            this.logger.success(`AI returned ${updates.length} files for update.`);
            return updates;
        } catch (e) {
            const errorSnippet = generatedText.substring(0, 200).replace(/\n/g, ' ');
            this.logger.error(`Failed to parse AI output as JSON: ${e.message}`);
            throw new Error(`Invalid JSON structure received from AI: ${errorSnippet}...`);
        }
    }
}

class EvolutionEngine {
    constructor(config) {
        this.config = {
            OWNER: 'craighckby-stack',
            REPO: 'ai-scaffold',
            SOURCE_BRANCH: 'main',
            VERSION: 18,
            ...config
        };
        this.logger = new Logger(this.config.LOG_ELEMENT_ID || 'evolution-log-output');
        this.github = null;
        this.gemini = null;
        this.db = null;
    }

    async initialize() {
        this.logger.log("Initializing Dalek Linear Evolution System v18...");
        
        const githubToken = this.config.GITHUB_TOKEN || window.prompt("Enter GitHub PAT:");
        const geminiKey = this.config.GEMINI_KEY || window.prompt("Enter Gemini API Key:");

        if (!githubToken) throw new Error("Initialization failed: GitHub Token missing.");
        if (!geminiKey) throw new Error("Initialization failed: Gemini Key missing.");
        
        this.github = new GitHubClient(githubToken, this.config.OWNER, this.config.REPO, this.logger);
        this.gemini = new GeminiClient(geminiKey, this.config.MODEL, this.logger);

        if (typeof firebase !== 'undefined' && this.config.FIREBASE_CONFIG) {
            try {
                 firebase.initializeApp(this.config.FIREBASE_CONFIG);
                 this.db = firebase.firestore();
                 this.logger.log("Firebase persistence enabled.", 'SETUP');
            } catch(e) {
                 this.logger.warn(`Firebase init failed: ${e.message}. Continuing without persistence.`);
            }
        }

        this.logger.success("System initialized successfully.");
    }

    async runEvolutionCycle(cycle = 1, totalCycles = 5) {
        this.logger.log(`--- Starting Evolution Cycle ${cycle}/${totalCycles} ---`, 'CYCLE');

        const evolutionBranch = `dalek-evolution-${Date.now()}`;
        const currentBranch = this.config.SOURCE_BRANCH;

        try {
            // 1. Establish Timeline
            const baseRef = await this.github.getReference(`heads/${currentBranch}`);
            const baseSha = baseRef.object.sha;
            await this.github.createBranch(evolutionBranch, baseSha);
            this.logger.success(`Evolution established on branch: ${evolutionBranch}`);

            // 2. Code Scanning
            const treeData = await this.github.getTree(baseSha, true);
            const filesToAnalyze = treeData.tree
                .filter(item => 
                    item.type === 'blob' && 
                    !item.path.includes('.git') && 
                    !item.path.includes('node_modules') &&
                    item.size < 1048576 // 1MB limit for AI context safety
                )
                .map(item => item.path);

            this.logger.log(`Found ${filesToAnalyze.length} files for analysis.`);
            
            const fileContents = {};
            await Promise.all(filesToAnalyze.map(async (path) => {
                fileContents[path] = await this.github.getFileContent(path, evolutionBranch);
            }));

            // 3. AI Analysis & Refinement
            const updates = await this.gemini.analyzeAndGenerateCode(fileContents);
            
            if (updates.length === 0) {
                 this.logger.warn("AI returned no modifications. Ending evolution early.");
                 return;
            }

            // 4. Commit Changes
            const commitMessage = `# Dalek Evolution v${this.config.VERSION} (Cycle ${cycle})\nAutomated refinement targeting optimization in ${updates.length} files.`;

            const newCommit = await this.github.commitChanges(evolutionBranch, commitMessage, updates);

            // 5. Persistence & Recursion
            this.logger.success(`Cycle ${cycle} complete. Changes committed.`);
            
            if (this.db) {
                await this.db.collection("evolution_history").add({
                    cycle,
                    timestamp: new Date(),
                    branch: evolutionBranch,
                    commitSha: newCommit.sha,
                    changes: updates.map(u => u.path)
                });
            }

            if (cycle < totalCycles) {
                this.config.SOURCE_BRANCH = evolutionBranch; 
                await this.runEvolutionCycle(cycle + 1, totalCycles);
            } else {
                this.logger.success(`--- Evolution Complete: ${totalCycles} Cycles Finalized. ---`, 'COMPLETE');
                this.logger.log(`Final branch: ${evolutionBranch}`);
            }

        } catch (error) {
            this.logger.error(`Evolution failed during Cycle ${cycle}: ${error.message}`);
            console.error(error);
        }
    }
}