# Dalek Linear Evolution - Recursive Commits v18

An autonomous code evolution system that iteratively improves codebases using AI-powered analysis and GitHub API integration.

## Features

- **Linear Evolution Cycles**: Runs through multiple refinement cycles to improve code quality
- **AI-Powered Analysis**: Uses Google Gemini API for intelligent code analysis and generation
- **GitHub Integration**: Automatically creates branches and commits improvements
- **Real-Time Feedback**: Provides detailed logging of each evolution cycle
- **Firebase Persistence**: Optionally stores evolution history in Firebase

## How to Use

1. **Open the Application**
   - Visit: https://craighckby-stack.github.io/ai-scaffold/
   - Or clone and run locally: open `index.html` in your browser

2. **Configure Your GitHub Token**
   - Generate a Personal Access Token at https://github.com/settings/tokens
   - Required permissions: `repo` (full control of private repositories)
   - Enter the token in the application

3. **Set Your Gemini API Key** (when prompted)
   - Get your API key from https://makersuite.google.com/app/apikey
   - The app will prompt for this when starting evolution

4. **Start Linear Evolution**
   - Click "Initiate 5-Cycle Evolution" (or 20-cycle in the full version)
   - Watch as the system:
     - Creates a new branch
     - Scans current codebase
     - Analyzes code with AI
     - Generates improvements
     - Commits changes to the branch

## Architecture

```
┌─────────────────────────────────────────────┐
│         Dalek Linear Evolution            │
└─────────────────────────────────────────────┘
                    │
                    ├─► GitHub API
                    │   ├─ Create Branch
                    │   ├─ Fetch Files
                    │   ├─ Commit Changes
                    │   └─ Read Repository
                    │
                    ├─► Gemini AI
                    │   ├─ Analyze Code
                    │   ├─ Identify Issues
                    │   ├─ Generate Fixes
                    │   └─ Optimize Architecture
                    │
                    └─► Firebase (Optional)
                        ├─ Store Evolution History
                        └─ Track Iterations
```

## Evolution Process

Each cycle performs:

1. **Timeline Establishment**: Creates a dedicated branch for evolution
2. **Code Scanning**: Reads and parses all project files
3. **AI Analysis**: 
   - Analyzes code structure
   - Identifies bugs and issues
   - Finds optimization opportunities
4. **Refinement**: Generates improved code for each file
5. **Commit**: Pushes changes to the evolution branch
6. **Repeat**: Continues to next cycle

## Configuration

### Firebase Setup (Optional)

To enable evolution history persistence:

```javascript
firebase.initializeApp({
  apiKey: "YOUR_FIREBASE_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID"
});
```

### Repository Structure

Ensure your repository is the target (set in code):
- Owner: `craighckby-stack`
- Name: `ai-scaffold`

## Security Notes

- **GitHub Token**: Never share or commit your token
- **Firebase Config**: Keep Firebase config private
- **API Keys**: Use environment variables for production

## Browser Compatibility

- Chrome/Edge 90+
- Firefox 88+
- Safari 14+

## License

This project is part of the Dalek Khan AGI research initiative.

## Contributing

Evolution is autonomous. Human intervention only required for:
- Initial token configuration
- Monitoring logs
- Stopping if needed

## Version

**v18** - Linear Evolution System
- Multi-cycle refinement
- Real-time logging
- Branch management
- Firebase persistence

---

**Built with ❤️ by Dalek Khan AGI**
