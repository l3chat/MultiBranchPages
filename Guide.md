# Complete GitHub Pages Multi-Branch Setup Guide

This comprehensive guide will walk you through setting up GitHub Pages with GitHub Actions to automatically deploy your static HTML files from three branches (main, dev, test) using only the GitHub web interface.

## Prerequisites

- A GitHub repository with static HTML files
- Three branches already created: `main`, `dev`, and `test`
- Repository admin access

## Part 1: Enable GitHub Actions (Web Interface)

### Step 1: Navigate to Actions Settings

1. Open your GitHub repository in a web browser
2. Click on the **Settings** tab (located at the top of the repository page)
3. In the left sidebar, scroll down and click on **Actions**
4. Click on **General** under the Actions section

### Step 2: Configure Actions Permissions

1. Under **Actions permissions**, select one of these options:
   - **Allow all actions and reusable workflows** (recommended for simplicity)
   - **Allow GitHub Actions created by GitHub, and select non-GitHub actions and reusable workflows**

2. Under **Workflow permissions**, select:
   - **Read and write permissions**
   - Check the box for **Allow GitHub Actions to create and approve pull requests**

3. Click **Save** at the bottom of the page

## Part 2: Enable GitHub Pages (Web Interface)

### Step 1: Navigate to Pages Settings

1. Still in the **Settings** tab of your repository
2. In the left sidebar, scroll down and click on **Pages**

### Step 2: Configure Pages Source

1. Under **Source**, select **GitHub Actions** from the dropdown menu
2. You should see a message: "Use GitHub Actions to deploy from any branch"
3. No need to click Save - this setting is applied automatically

## Part 3: Create GitHub Actions Workflow (Web Interface)

### Step 1: Navigate to Actions Tab

1. Click on the **Actions** tab at the top of your repository
2. You'll see the Actions dashboard

### Step 2: Create New Workflow

1. Click on **New workflow** button (green button on the right)
2. You'll see a page with workflow templates
3. Click on **set up a workflow yourself** (link in the top section)
4. This will create a new file at `.github/workflows/main.yml`

### Step 3: Replace Default Workflow Content

1. Delete all the default content in the editor
2. Replace it with the following workflow code:

```yaml
# ===================================================================
# GitHub Actions Workflow for Multi-Branch GitHub Pages Deployment
# ===================================================================
# 
# This workflow automatically deploys static HTML files from multiple
# branches (main, dev, test) to GitHub Pages with branch-specific URLs
#
# Branch URLs after deployment:
# - main:  https://username.github.io/repository-name/
# - dev:   https://username.github.io/repository-name/dev/
# - test:  https://username.github.io/repository-name/test/
# ===================================================================

name: Deploy Multi-Branch to GitHub Pages

# ===================================================================
# TRIGGER CONDITIONS
# ===================================================================
# This workflow will run when:
# 1. Code is pushed to main, dev, or test branches
# 2. Workflow is manually triggered from GitHub Actions tab
on:
  push:
    branches: [ main, dev, test ]  # Only these branches trigger deployment
  workflow_dispatch:               # Allows manual triggering

# ===================================================================
# PERMISSIONS CONFIGURATION
# ===================================================================
# These permissions are required for GitHub Pages deployment
permissions:
  contents: read    # Read repository contents
  pages: write      # Write to GitHub Pages
  id-token: write   # Write identity token (for secure deployment)

# ===================================================================
# CONCURRENCY CONTROL
# ===================================================================
# Prevents multiple deployments from running simultaneously
# Each branch gets its own deployment group to avoid conflicts
concurrency:
  group: "pages-${{ github.ref_name }}"  # Unique group per branch
  cancel-in-progress: false               # Don't cancel running deployments

# ===================================================================
# JOBS DEFINITION
# ===================================================================
jobs:
  # Main deployment job
  build-and-deploy:
    runs-on: ubuntu-latest  # Use Ubuntu runner (free for public repos)
    
    # ===================================================================
    # ENVIRONMENT CONFIGURATION
    # ===================================================================
    # Links this job to GitHub Pages environment
    # Requires environment permissions to be configured in Settings > Environments
    environment:
      name: github-pages                              # Must match environment name
      url: ${{ steps.deployment.outputs.page_url }}  # Will contain deployment URL
    
    steps:
      # ===================================================================
      # STEP 1: DOWNLOAD REPOSITORY CODE
      # ===================================================================
      # Downloads all repository files to the runner
      - name: Checkout repository
        uses: actions/checkout@v4  # Official GitHub action for code checkout
        with:
          fetch-depth: 0          # Download complete git history (optional)
        
      # ===================================================================
      # STEP 2: CONFIGURE GITHUB PAGES
      # ===================================================================
      # Sets up GitHub Pages configuration and validates settings
      - name: Setup Pages configuration
        uses: actions/configure-pages@v4  # Official GitHub Pages configuration action
        
      # ===================================================================
      # STEP 3: DETERMINE DEPLOYMENT STRATEGY
      # ===================================================================
      # Decides how to deploy based on which branch triggered the workflow
      - name: Determine deployment path and URL
        id: deploy-info                    # This ID is used to reference outputs later
        run: |
          # Print current branch for debugging
          echo "Current branch: ${{ github.ref_name }}"
          
          # Check if this is the main branch
          if [[ "${{ github.ref_name }}" == "main" ]]; then
            echo "Deploy main branch to root"
            echo "deploy_path=." >> $GITHUB_OUTPUT        # Deploy to root directory
            echo "is_main=true" >> $GITHUB_OUTPUT         # Flag for main branch
          else
            echo "Deploy ${{ github.ref_name }} branch to subdirectory"
            echo "deploy_path=${{ github.ref_name }}" >> $GITHUB_OUTPUT  # Deploy to branch folder
            echo "is_main=false" >> $GITHUB_OUTPUT                       # Flag for other branches
          fi
      
      # ===================================================================
      # STEP 4: PREPARE FILES FOR DEPLOYMENT
      # ===================================================================
      # Creates the correct directory structure for deployment
      - name: Prepare deployment files
        run: |
          echo "Preparing files for deployment..."
          
          # Create temporary directory for deployment files
          mkdir -p deployment_temp
          
          # ===================================================================
          # MAIN BRANCH DEPLOYMENT (to root)
          # ===================================================================
          if [[ "${{ steps.deploy-info.outputs.is_main }}" == "true" ]]; then
            echo "Setting up main branch deployment"
            # Copy all files to deployment directory
            cp -r * deployment_temp/ 2>/dev/null || true
            # Remove nested deployment_temp to prevent recursion
            rm -rf deployment_temp/deployment_temp 2>/dev/null || true
            # Remove .github directory (not needed in deployment)
            rm -rf deployment_temp/.github 2>/dev/null || true
            
          # ===================================================================
          # OTHER BRANCHES DEPLOYMENT (to subdirectories)
          # ===================================================================
          else
            echo "Setting up ${{ github.ref_name }} branch deployment"
            # Create subdirectory for this branch
            mkdir -p deployment_temp/${{ github.ref_name }}
            # Copy all files to branch subdirectory
            cp -r * deployment_temp/${{ github.ref_name }}/ 2>/dev/null || true
            # Clean up nested directories to prevent recursion
            rm -rf deployment_temp/${{ github.ref_name }}/deployment_temp 2>/dev/null || true
            rm -rf deployment_temp/${{ github.ref_name }}/.github 2>/dev/null || true
            
            # ===================================================================
            # CREATE NAVIGATION INDEX PAGE
            # ===================================================================
            # Create a simple index.html at root for easy navigation between branches
            cat > deployment_temp/index.html << 'EOF'
          <!DOCTYPE html>
          <html lang="en">
          <head>
              <meta charset="UTF-8">
              <meta name="viewport" content="width=device-width, initial-scale=1.0">
              <title>Branch Deployments</title>
              <style>
                  body { 
                      font-family: Arial, sans-serif; 
                      max-width: 600px; 
                      margin: 50px auto; 
                      padding: 20px; 
                  }
                  .branch { 
                      background: #f5f5f5; 
                      padding: 15px; 
                      margin: 10px 0; 
                      border-radius: 5px; 
                  }
                  a { 
                      color: #0366d6; 
                      text-decoration: none; 
                  }
                  a:hover { 
                      text-decoration: underline; 
                  }
              </style>
          </head>
          <body>
              <h1>Available Branch Deployments</h1>
              <div class="branch">
                  <h3>Development Branches</h3>
                  <p><a href="./dev/">Dev Branch</a> - Development environment</p>
                  <p><a href="./test/">Test Branch</a> - Testing environment</p>
              </div>
              <p><em>Note: This page appears when non-main branches are deployed.</em></p>
          </body>
          </html>
          EOF
          fi
          
          # ===================================================================
          # DEBUGGING: SHOW DEPLOYMENT STRUCTURE
          # ===================================================================
          # List files for debugging purposes (visible in Actions logs)
          echo "Deployment structure:"
          ls -la deployment_temp/
          
      # ===================================================================
      # STEP 5: UPLOAD FILES TO GITHUB PAGES
      # ===================================================================
      # Packages files for GitHub Pages deployment
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3  # Official GitHub Pages upload action
        with:
          path: ./deployment_temp                # Directory containing files to deploy
          
      # ===================================================================
      # STEP 6: DEPLOY TO GITHUB PAGES
      # ===================================================================
      # Actually deploys the files to GitHub Pages
      - name: Deploy to GitHub Pages
        id: deployment                    # ID used for getting deployment URL
        uses: actions/deploy-pages@v4     # Official GitHub Pages deployment action

  # ===================================================================
  # NOTIFICATION JOB
  # ===================================================================
  # Runs after deployment to provide status information
  notify-deployment:
    needs: build-and-deploy              # Wait for deployment to complete
    runs-on: ubuntu-latest               # Use Ubuntu runner
    if: always()                         # Run even if deployment failed
    
    steps:
      # Provide deployment status and URLs
      - name: Deployment status notification
        run: |
          echo "=== Deployment Summary ==="
          echo "Branch: ${{ github.ref_name }}"
          echo "Status: ${{ needs.build-and-deploy.result }}"
          echo "Repository: ${{ github.repository }}"
          
          # ===================================================================
          # SUCCESS NOTIFICATION
          # ===================================================================
          if [[ "${{ needs.build-and-deploy.result }}" == "success" ]]; then
            if [[ "${{ github.ref_name }}" == "main" ]]; then
              echo "🚀 Main branch successfully deployed!"
              echo "URL: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/"
            else
              echo "🚀 ${{ github.ref_name }} branch successfully deployed!"
              echo "URL: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}/${{ github.ref_name }}/"
            fi
          # ===================================================================
          # FAILURE NOTIFICATION
          # ===================================================================
          else
            echo "❌ Deployment failed for ${{ github.ref_name }} branch"
            echo "Check the workflow logs for details"
            echo "Common issues:"
            echo "- Workflow file missing in branch"
            echo "- Environment permissions not configured"
            echo "- GitHub Pages not enabled"
          fi
```

### Step 4: Configure Workflow File

1. In the file name field at the top, change `main.yml` to `deploy.yml`
2. Review the workflow code you just pasted
3. Make sure all the code is properly formatted (GitHub will show syntax highlighting)
4. **IMPORTANT**: Ensure the folder name is `.github` (with lowercase 'g'), not `.GitHub`

### Step 5: Commit the Workflow

1. Scroll down to the **Commit new file** section
2. In the commit message field, enter: `Add GitHub Pages deployment workflow`
3. In the description field (optional), enter: `Automated deployment for main, dev, and test branches`
4. Select **Commit directly to the main branch**
5. Click **Commit new file**

### Step 6: Copy Workflow to All Branches

**CRITICAL**: The workflow file must exist in ALL branches to work properly. GitHub Actions only runs workflows that exist in the branch being pushed to.

#### Method 1: Using Pull Requests (Recommended)

**Copy to dev branch:**
1. Stay in the main branch
2. Click **"Contribute"** → **"Open pull request"**
3. Set **base:** `dev` ← **compare:** `main`
4. Add title: `Add workflow to dev branch`
5. Click **"Create pull request"** → **"Merge pull request"**

**Copy to test branch:**
1. Repeat the above steps but set **base:** `test` ← **compare:** `main`

#### Method 2: Manual Copy (Alternative)

**For dev branch:**
1. Switch to `dev` branch
2. Create new file: `.github/workflows/deploy.yml`
3. Copy the entire workflow code from main branch
4. Commit the file

**For test branch:**
1. Switch to `test` branch  
2. Create new file: `.github/workflows/deploy.yml`
3. Copy the entire workflow code from main branch
4. Commit the file

## Part 4: Configure Environment Permissions

### Step 1: Navigate to Environments

1. Go to **Settings** tab of your repository
2. In the left sidebar, find and click **"Environments"**
3. You should see the **github-pages** environment (created automatically when using GitHub Pages)

### Step 2: Configure Branch Access

1. Click on the **"github-pages"** environment
2. Under **"Deployment branches"**, you'll see a dropdown
3. Change it from **"Protected branches only"** to **"Selected branches"**
4. Click **"Add deployment branch rule"**
5. Add the following branch patterns one by one:
   - `main`
   - `dev`  
   - `test`

### Step 3: Save Environment Settings

1. After adding all three branch rules, the settings save automatically
2. You should now see all three branches listed under "Deployment branches"

## Part 5: Test the Deployment (Web Interface)

### Step 1: Verify Workflow Creation

1. After committing, you'll be redirected to the Actions tab
2. You should see your new workflow listed as **Deploy Multi-Branch to GitHub Pages**
3. The workflow should start running automatically (since you just pushed to main)

### Step 2: Monitor First Deployment

1. Click on the running workflow to view its progress
2. You'll see the job **build-and-deploy** running
3. Click on the job to see detailed logs
4. Wait for it to complete (should take 1-3 minutes)

### Step 3: Check GitHub Pages Deployment

1. Go to **Settings** > **Pages**
2. You should see a green checkmark and your site URL
3. Click on **Visit site** to view your deployed main branch

### Step 4: Test Branch Deployments

**IMPORTANT**: Before testing dev/test branches, ensure the workflow file exists in those branches (see Part 3, Step 6).

1. Navigate to your **dev** branch:
   - Click on the branch dropdown (usually shows "main")
   - Select **dev** branch

2. Verify the workflow file exists:
   - Check that `.github/workflows/deploy.yml` exists in the dev branch
   - If not, follow the instructions in Part 3, Step 6 to copy it

3. Make a small change to trigger deployment:
   - Click on any HTML file (e.g., `index.html`)
   - Click the **Edit** button (pencil icon)
   - Add a comment or change some text
   - Scroll down and commit the change

4. Go to **Actions** tab to see the new workflow running for the dev branch

5. Once complete, visit your site URL with `/dev/` added to test the dev deployment

6. Repeat for the **test** branch

## Part 6: Understanding Your Deployment URLs

After successful setup, your branches will be accessible at:

- **Main branch**: `https://yourusername.github.io/yourrepo/`
- **Dev branch**: `https://yourusername.github.io/yourrepo/dev/`
- **Test branch**: `https://yourusername.github.io/yourrepo/test/`

## Part 7: Managing Deployments (Web Interface)

### Viewing Deployment History

1. **Actions Tab**: See all workflow runs and their status
2. **Settings > Pages**: View current deployment status and URL
3. **Environments**: Go to repository main page, click **Environments** to see deployment history

### Manual Triggering

1. Go to **Actions** tab
2. Click on **Deploy Multi-Branch to GitHub Pages** workflow
3. Click **Run workflow** button
4. Select the branch you want to deploy
5. Click **Run workflow**

### Monitoring and Troubleshooting

1. **Check Workflow Logs**:
   - Actions tab > Click on workflow run > Click on job name
   - Review step-by-step logs for errors

2. **Common Issues**:
   - **Permission errors**: Verify Actions permissions in Settings > Actions
   - **Pages not enabled**: Check Settings > Pages configuration
   - **File not found**: Ensure your HTML files exist in the branch

## Part 8: Advanced Configuration Options

### Adding Environment Variables

1. Go to **Settings** > **Secrets and variables** > **Actions**
2. Click **New repository secret**
3. Add secrets that your workflow can use

### Branch Protection Rules

1. Go to **Settings** > **Branches**
2. Click **Add rule**
3. Configure protection for your main branch

### Custom Domain (Optional)

1. Go to **Settings** > **Pages**
2. Under **Custom domain**, enter your domain
3. Configure DNS settings as instructed

## Troubleshooting Common Issues

### Issue 1: Workflow Not Running
**Solutions**: 
- Check that GitHub Actions is enabled in Settings > Actions
- **Verify workflow file exists in the branch you're pushing to**
- Ensure folder name is `.github` (lowercase), not `.GitHub`

### Issue 2: "Branch is not allowed to deploy" Error
**Solution**: Configure environment permissions in Settings > Environments > github-pages > Deployment branches

### Issue 3: Permission Denied
**Solution**: Verify workflow permissions are set to "Read and write permissions"

### Issue 4: Pages Not Updating
**Solution**: Check that source is set to "GitHub Actions" in Settings > Pages

### Issue 5: 404 Error on Branch URLs
**Solutions**: 
- Ensure the branch has been pushed to and the workflow completed successfully
- Check that the workflow file exists in all branches (main, dev, test)

### Issue 6: Workflow Only Works for Main Branch
**Solution**: Copy the workflow file (`.github/workflows/deploy.yml`) to all branches using Pull Requests or manual copying

## Security Best Practices

1. **Review Permissions**: Only grant necessary permissions to workflows
2. **Protect Main Branch**: Use branch protection rules for production deployments  
3. **Monitor Activity**: Regularly check Actions logs for unusual activity
4. **Keep Actions Updated**: GitHub will notify about action updates

This completes your GitHub Pages multi-branch deployment setup using only the web interface!