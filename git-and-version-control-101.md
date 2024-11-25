# Git and Version Control 101: A Beginner's Handbook

## 1. Understanding Version Control

### What is Version Control?
Version control is a system that records changes to files over time, allowing you to:
- Track modifications and compare file versions
- Revert files or projects to previous states
- Review changes made over time
- Collaborate with multiple people on the same project
- Identify who made specific changes and when

### Why is Version Control Necessary?
Consider these common scenarios:
1. **Multiple Versions Problem**: 
   - `final_report.doc`
   - `final_report_v2.doc`
   - `final_report_v2_final.doc`
   - `final_report_v2_final_REALLY_FINAL.doc`

2. **Collaboration Challenges**:
   - Multiple team members working on the same code
   - Conflicting changes
   - No way to merge work efficiently

3. **Business Requirements**:
   - Audit trails for changes
   - Ability to roll back problematic changes
   - Disaster recovery
   - Code review processes

## 2. Introduction to Git

### What is Git?
Git is a distributed version control system created by Linus Torvalds in 2005. It's:
- Fast and efficient
- Distributed (every developer has a full copy)
- Supports non-linear development (branching)
- Free and open source

### Key Concepts
1. **Repository (Repo)**:
   - A container for your project
   - Contains all files and version history

2. **Commit**:
   - A snapshot of changes
   - Includes a unique identifier (hash)
   - Contains author information and timestamp

3. **Branch**:
   - A separate line of development
   - Allows parallel work without affecting main code

4. **Remote**:
   - A repository hosted on the internet/network
   - Enables collaboration with others

## 3. Hands-on Practice

### Setup and Configuration

1. **Install Git**:
   ```bash
   # For Ubuntu/Debian
   sudo apt-get update
   sudo apt-get install git

   # For CentOS/RHEL
   sudo yum install git
   ```

2. **Configure Git**:
   ```bash
   git config --global user.name "Your Name"
   git config --global user.email "your.email@example.com"
   ```

### Basic Git Operations

1. **Creating a New Repository**:
   ```bash
   # Create a new directory
   mkdir my_first_repo
   cd my_first_repo

   # Initialize git repository
   git init
   ```

2. **Making Your First Commit**:
   ```bash
   # Create a file
   echo "Hello, Git!" > hello.txt

   # Stage the file
   git add hello.txt

   # Commit the file
   git commit -m "My first commit"
   ```

3. **Checking Status and History**:
   ```bash
   # Check repository status
   git status

   # View commit history
   git log
   ```

### Practice Exercise 1: Basic Git Workflow
1. Create a new directory called `git-practice`
2. Initialize it as a git repository
3. Create a file called `README.md` with some text
4. Stage and commit the file
5. Modify the file
6. Check the status to see the changes
7. Stage and commit the changes

### Practice Exercise 2: Working with Branches
```bash
# Create and switch to a new branch
git checkout -b feature-branch

# Make some changes
echo "New feature" > feature.txt
git add feature.txt
git commit -m "Add new feature"

# Switch back to main branch
git checkout main

# Merge the feature branch
git merge feature-branch
```

### Practice Exercise 3: Remote Repositories
```bash
# Clone a remote repository
git clone https://github.com/example/repo.git

# Add a remote repository to existing local repo
git remote add origin https://github.com/example/repo.git

# Push changes to remote
git push origin main

# Pull changes from remote
git pull origin main
```

## 4. Common Git Commands Reference

### Basic Commands
- `git init`: Initialize a new repository
- `git clone`: Copy a repository
- `git add`: Stage changes
- `git commit`: Record changes
- `git status`: Check repository status
- `git log`: View history

### Branch Operations
- `git branch`: List branches
- `git checkout`: Switch branches
- `git merge`: Combine branches
- `git branch -d`: Delete branch

### Remote Operations
- `git remote`: Manage remotes
- `git push`: Upload changes
- `git pull`: Download changes
- `git fetch`: Get remote changes

## 5. Best Practices

### Commit Messages
- Write clear, descriptive messages
- Use present tense ("Add feature" not "Added feature")
- Keep first line under 50 characters
- Provide detailed description if needed

### Branching Strategy
- Keep `main` branch stable
- Create feature branches for new work
- Delete branches after merging
- Use meaningful branch names

### General Tips
- Commit often and in logical chunks
- Pull before pushing
- Review changes before committing
- Keep repositories clean

## 6. Troubleshooting Common Issues

### Issue 1: Merge Conflicts
```bash
# When you encounter a merge conflict:
1. Open the conflicted files
2. Look for conflict markers (<<<<<<, =======, >>>>>>>)
3. Resolve the conflicts manually
4. Stage the resolved files
5. Complete the merge with commit
```

### Issue 2: Undoing Changes
```bash
# Discard changes in working directory
git checkout -- filename

# Unstage changes
git reset HEAD filename

# Undo last commit
git reset --soft HEAD^
```

## 7. Additional Resources
- [Git Official Documentation](https://git-scm.com/doc)
- [GitHub Learning Lab](https://lab.github.com/)
- [Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf)