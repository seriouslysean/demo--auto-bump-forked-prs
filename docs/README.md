

# Automating Version Bumps using GitHub Actions

![Half closed macbook pro with an orange, blue and pink wavy background and the keyboard reflecting on the screen. Photo by Eliezer Pujols on Unsplash.](./half-closed-laptop.jpg)

## The Problem

In many projects I've worked on, managing versions has been a little tricky, both conceptually and also in practice. There are a few ways I've tried in the past:

- Update the version in each PR
- Update the version at certain intervals with a set amount of changes
- Update the version in a separate PR after each commit

There have been challenges with each method; what if some of the PRs are dependent on others, or what if the order of merges changes? You would need to change the version in your PR and it ends up being a huge hassle, not to mention a new commit might remove pre-existing approvals on the PR.

I've created this demo repo that I used as a sandbox to figure out a method to auto bump the version from both a repo PR and a forked PR merge event, to a protected `main` branch. As `main` is protected, workflows can't typically push to it within the workflow after bumping the version unless they have the privilege to bypass the branch controls.

One approach that is sometimes suggested is to create a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) (or PAT) with rights to bypass branch controls. Then, share it as a secret in the GitHub org/repo so that workflow files can pull the PAT in and use it to authenticate with GitHub, which will allow overriding branch protections.

This level of access introduces security concerns, though; anyone with the rights to push code that triggers these workflows could potentially rewrite the workflow code to push to any branch. Additionally, it does not enable the version bump to be triggered by merges into `main` that come from forked repos due to the context that workflows run in. Forked PRs do not have access to secrets for security reasons.

## The Solution

Two workflows can be used to bypass the restriction of secrets not being available to workflow runs when triggered by a forked PR. In order to do this, we need to break the steps out into discrete steps:

### Workflow One: Prepare the Version Bump

This workflow's job is to:

1. Make sure that the PR was closed and merged to `main`
2. Make sure that ONE version bump label exists on the PR
3. Save the bump type to an artifact, using the id of this workflow run

### Workflow Two: Bump Version

This workflow's job is to:

1. Run whenever Workflow One reaches `completed` status
2. Check if the previous workflow was `successful`
3. Download the artifact from the previous workflow, using the id of the `workflow_run` trigger
4. Use the artifact data to do the version bump
5. Create the release using the next version after bump

#### Notes

- Workflow One runs in the context of the forked PR and _does not_ have access to secrets
- Workflow Two runs in the context of this repo and _does_ have access to secrets
- The artifact uploaded only becomes available to the API after the workflow completes as far as I can tell, thus we need to check for the existence in Workflow Two
- I only pass the TYPE of bump and not the version, because of the risk of workflows running out of order. This may be helped with [concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency), but would require testing, as I wouldn't want to lose a bump due to cancellation

## Other Solutions

I've looked at many blog posts on the matter, and tried out almost every example out there on how to get the auto bump logic to work. There is a longstanding GitHub discussion on the issue, too, [GitHub Discussion](https://github.com/orgs/community/discussions/25305).

### 1. Let PRs from the Repo Auto Bump via Workflow (No Forked PR Support)

This solution requires a Personal Access Token (PAT) to be generated and added to the repo secrets in GitHub. However, forked PRs do not have access to these secrets, causing this method to fail for forked PRs.

This is the method currently used by [Cuttle](https://github.com/cuttle-cards/cuttle), a side project I work on. The workflow that handles the bump logic is here, [Bump Version Workflow](https://github.com/cuttle-cards/cuttle/blob/1288fd06235a975e77b9ecc53b728831a30f253f/.github/workflows/bump-version.yml).

### 2. Merged PRs Create a new Version Bump PR to be Auto Merged

This solution doesn't necessarily require a PAT, but it introduces a security risk due to the nature of the auto-merge feature. Auto-merge is a repository-wide setting, meaning it applies to all PRs, not just the version bump ones (which is a huge risk, in my opinion). If it's enabled, any PR that passes all status checks would be automatically merged, which might not be the desired behavior, especially for PRs that require manual review.

Additionally, this method can clutter up the PR tab in GitHub with numerous version bump PRs. If branch protection rules are in place, they might prevent the auto-merge from working as expected. For example, if a rule requires a certain number of manual reviews before a PR can be merged, the auto-merge would not be able to merge the PR until those reviews are provided.

### 3. Use a GitHub App to Handle the Version Bumping

This method involves creating and maintaining a GitHub app to handle the version bumping. While it has a higher barrier to entry, it offers more flexibility and can support version bumping when forked PRs are merged. The app operates with its own set of permissions, independent of the repository's secrets. However, even with its own permissions, the app must still adhere to any branch protection rules in place.

## Final Thoughts

I've [successfully implemented this method on an open-source project](https://github.com/cuttle-cards/cuttle/pull/830) I contribute to, [Cuttle](https://cuttle.cards), big thanks to [Ryan Emberling](https://github.com/itsalaidbacklife) for helping me test everything (and letting me go rogue to merge it in the project).

In the end, I've learned a lot about GitHub actions, solved a problem that I previously thought was not able to be solved, and made the contributor experience in Cuttle much better for the effort, so I consider that a resounding win.

Enjoy!

-----

## Links

- [GitHub Discussion](https://github.com/orgs/community/discussions/25305#discussioncomment-7768235)
- [LinkedIn Article on Approach](https://www.linkedin.com/pulse/automating-version-bumps-using-github-actions-sean-kennedy-3redf%3FtrackingId=CrBB6hpMRvmwLK8mi8tJ2g%253D%253D/?trackingId=CrBB6hpMRvmwLK8mi8tJ2g%3D%3D)
