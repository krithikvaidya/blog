---
layout: post
title: "Git's Data Model: Explanation and Hands-on"
author: "Krithik Vaidya"
tags: [git, version-control]
image: /Internals-of-Git/git-logo.jpg
---

## Introduction to Version Control Systems

Version control systems are a category of software tools that helps record changes to the files of a software project by keeping a track of modifications done to the code. Versioning your code is a priceless process, especially when you have multiple developers working on a single application, because it allows them to work parallelly on the same code base. Without version control, developers of large software projects will eventually overwrite code changes that someone else may have completed without even realizing it. So It’s an invaluable tool for tracking the history of a software project, seeing what other people have changed, as well as resolving conflicts in concurrent development. Even when you’re working by yourself, it can let you look at old snapshots of a project, keep a log of why certain changes were made, work on parallel branches of development, and much more!

[Git](https://git-scm.com/) is currently the most popular version control tool. 

Now, we will approach Git from a bottom-up manner, to get a better idea about the underlying design and internals of Git, i.e., Git's data model. Once the underlying data model is understood, the commands that we use to interact with a Git repository can be better understood, in terms of how they manipulate the underlying data model.

## Git's data model

_Warning_: Brace yourself, a lot of jargon is coming your way!

Git's data model is based on a Graph-like structure, specifically a **Directed Acyclic Graph (DAG)**. A DAG is a graph that is *directed* (every edge between two nodes is unidirectional, i.e., points toward one of the nodes), and *acyclic* (it is impossible to find a path starting at any given node such that on traversing the path, we end up back at the starting node). 

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/1.png" alt="Git DAG"><figcaption>A sample Directed Acyclic Graph. In Git, each node represents an <em>Object</em></figcaption></figure>

There is some more terminology we need to understand before proceeding:-

**Blob**: 
The simplest object in git, just a bunch of bytes. This is usually a file.

**Tree**: 
In Git, a directory is called a “tree”, and it maps a name (i.e. a _hash_) to a set of blobs and/or trees. (i.e. similar to a folder, which can contain files as well as other folders). So a tree can point to blobs as well as other trees.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/tree.png" alt="Tree pointing to blobs"><figcaption>A tree pointing to two blobs</figcaption></figure>

Or another way to view a tree: (the top level tree contains another tree, *foo*, and a blob, *baz.txt*)

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/tree_vis.png" alt="Alternate tree"></figure>

Note:- When we say that a node points to another node in the DAG, it means that the node depends on the node it is pointing to, and should not exist without it.

**Snapshot (or Commit)**:
A snapshot or a commit stores a reference to the top level tree that is being tracked. Each commit has its own hash and saves metadata associated with the commit such as the commit message, timestamp and info of the user making the commit.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/commit.png" alt="A commit"><figcaption>A commit</figcaption></figure>

**Hash**:
You may see strange strings such as *de9f2c7fd25e1b3afad3e85a0bd17d9b100db4b3* or its short form (just the first few characters of the string) everywhere in Git. This is **SHA1 hash** of a Git object (a Git object refers to either a tree, a blob or a commit). A hash can be produced from any object.
The hash of an object basically identifies it uniquely. It is 40 characters long but only few (usually the first seven) are ordinarily enough to identify a commit.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/hash.png" alt="Example Hashes"><figcaption>Example Hashes</figcaption></figure>

**References**
Now, all snapshots can be identified by their SHA-1 hash. That’s inconvenient, because humans aren’t good at remembering strings of 40 hexadecimal characters. Git’s solution to this problem is human-readable names for SHA-1 hashes, called “references”. References are basically pointers to commits with human readable names.

References are mutable (can be updated to point to a new commit). For example, the master reference usually points to the latest commit in the main branch of development. Similarly, there is the special HEAD reference that points to the latest commit that our work is currently based off on.

Other than these references, there are a lot more references stored for every git repository. You can navigate to the .git/refs directory inside a Git repository to view all the references present for that repository.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/refs.png" alt="The refs folder"><figcaption>Git stores four types of refs</figcaption></figure>

**Repositories**
Roughly, a git repository is the collection of all data objects and references.

On disk, all that Git stores is objects and references! All git commands map to some manipulation of the commit history graph by adding objects and adding/updating references.

**Modelling Git history as a timeline of commits**

In Git, a history is a directed acyclic graph (DAG) of snapshots. All this means is that each snapshot in Git refers to a set of “parents”, the snapshots that preceded it.

*Note:* It’s a set of parents rather than a single parent (as would be the case in a linear history) because a snapshot might descend from multiple parents, for example due to combining (merging) two parallel branches of development.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/commits_timeline.png" alt="Timeline of commits"><figcaption>Visualizing a Sample Commit History</figcaption></figure>

Each circle above corresponds to an individual commit, with the leftmost commit (labelled 'A' above) being the initial commit. The edges in the graph point toward the parent commit, not toward the child commit, to reflect how Git internally connects commits. Based off of commit 'B', a separate branch is created, which can represent development occuring parallely in the same repository. Again at commit 'C', another branch is created. Eventually, the new branches merge into one (at point 'D'), creating a merge commit. The merged branches and the original branch merge again at point 'E', creating another merge commit.

**Summary of Git's data model**:

To summarize, it may be instructive to see Git’s data model written down in pseudocode (all credits to [https://missing.csail.mit.edu](https://missing.csail.mit.edu) for the below pseudocode):

```
// a file is a bunch of bytes
type blob = array<byte>

// a directory contains named files and directories
type tree = map<string, tree | blob>

// a commit has parents, metadata, and the top-level tree
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}

// Objects and content-addressing
// An “object” is a blob, tree, or commit:

type object = blob | tree | commit

// In Git data store, all objects are content-addressed by their SHA-1 hash.

objects = map <string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]

// references

references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)

```

When Git objects (blobs, trees, references) reference other objects, they don’t actually contain them in their on-disk representation, but have a reference to them by their hash.

Apart from these features, Git also has a mechanism called the *staging area*, which contains all the files and folders based on which the next snapshot will be created, based on their states at the time of snapshot creation (i.e., when you *git add* files in your working directory and then *git commit* them).

#### A small hands-on

To better understand what we've learned, we can perform a small exercise. Can we see the contents of a specific file in a previous commit, by only traversing the *.git* directory and using some [plumbing](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain) commands? Let's start by opening the .git directory in our terminal and inspecting its contents. Feel free to follow along using any Git repository of your choice.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a1.png" alt=".git"></figure>

You should now be familiar with some of the contents of the directory (objects, refs, branches, etc.). If you're interested, you should explore the directory further.

We can see that there's a file named **HEAD**. On displaying it's contents, we see that - 

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a2.png" alt="HEAD contents"></figure>

It tells us that HEAD is currently pointing to the master branch, and the hash related to the master *reference* (talked about above) can be found in refs/heads/master!

On navigating to *refs/heads/* and inspecting the contents of file master, we see -

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a3.png" alt="master contents"></figure>

We know that the long string of cryptic characters can be only one thing, a SHA1 hash of some Git object! Git provides a plumbing command, *git cat-file*, to get all kinds of information related to a given hash that's part of our repo. Running *git cat-file* with the -t flag (tells us the type of object that is pointed to by the given hash), we get -

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a4.png" alt="cat file type"></figure>

The hash points to a commit! (Note that we didn't need to use all 40 characters of the hash, just the first few was sufficient). We know that the HEAD reference usually points to the latest commit in the master branch, so this makes sense. Using cat-file and the -p flag (pretty print), we can see all the information stored in the commit.

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a5.png" alt="cat file print"></figure>

We can see that the *tree* our commit points to is represented by the hash 123366691fea28ecc9c2dd672a686ea4fa3dd0b7. Using cat-file pretty-print on this, we see

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a6.png" alt="print tree"></figure>

All the blobs and tree that are 'stored' within this tree. This is basically all the files and folders in our latest commit! And running cat-file on one of the blobs shows us the file contents, as it were at that commit!

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a7.png" alt=".git"><figcaption>Revealing the contents of the blob with hash de1af..., which is of the file requirements.txt</figcaption></figure>

Finally, while inpecting the commit, we saw the it had another field called *parent*

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a8.png" alt="parent in commit"></figure>

What should the type of the hash of this parent be? Since parents of commits are other commits (as explained in the visualising commit history section),

<figure class="image" style="text-align: center; color: gray;"><img src="/blog/assets/img/Internals-of-Git/a9.png" alt="type of parent in commit"></figure>

It's the hash of a commit. Now, we can repeat the previous steps to see the contents of files and folders of the repository in that commit, and so on!

Hopefully this little exercise helped you get a better grip of Git's data model.

#### Conclusion

In this article, we've explored Git's data model, and understood terms such as blobs, references, trees, snapshots, objects, etc., in the context of Git. Then we visualized an example commit history and used our knowledge to traverse the *.git* directory and perform useful tasks. The important takeaway is that once the data model is understood, we gain a deeper understanding what various Git commands do in terms of manipulating the underlying data model.

#### References
[https://missing.csail.mit.edu](https://missing.csail.mit.edu)  
[https://jwiegley.github.io/git-from-the-bottom-up/](https://jwiegley.github.io/git-from-the-bottom-up/)