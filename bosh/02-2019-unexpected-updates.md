# Unexpected Updates When Using Compiled Releases

## Steps to Reproduce

1. Deploy a deployment (deployment A) using the following configuration:

   Deployment A contains:
     * Stemcell Y.X
     * Release X compiled against Stemcell Y.X

2. Upgrade the stemcell for the deployment (deployment A) without changing the
   release version

   Deployment A contains:
     * Stemcell Y.X+1
     * Release X compiled against Stemcell Y.X

3. Deploy a deployment (deployment B) using the following configuration:

   Deployment A contains:
     * Stemcell Y.X+1
     * Release X compiles against Stemcell Y.X

   Deployment B contains:
     * Stemcell Y.X+2
     * Release X compiled against Stemcell Y.X+2

4. Perform an update on the first deployment (deployment A) that should not
   update any of the existing instances i.e. scale up by 1 instance.

   At this phase you should see the entirety of the first deployment (deployment
   A) update unexpected. At the end of this deploy you will see the following
   configuration:

   Deployment A contains:
     * Stemcell Y.X+1
     * Release X compiles against Stemcell Y.X+2

   Deployment B contains:
     * Stemcell Y.X+2
     * Release X compiled against Stemcell Y.X+2

## Explanation

The core of this problem originates from how BOSH handles compiled packages that
do not have the underlying source package.

BOSH will always try to use compiled packages that match the stemcell that you
are deploying. Thus if you are deploying a stemcell of version Y.X, and you have
a package that is compiled against Y.X, BOSH will use that version of the
package. This can be seen in Step 1 of the reproduction.

Unfortunately, stemcells often are updated at a higher frequency than releases,
and we want to provide a solution that does not force users to generate compiled
releases every time that a stemcell changes.

Thus compiled packages are allowed to "float" against any major stemcell line. This
means that if you have a compiled release that has been compiled against a
stemcell with the version Y.X, you will still be able to successfully deploy
that compiled release against any stemcell in the Y line. This can be seen in
Step 2 of the reproduction.

One interesting facet of this is that BOSH prefers to use the latest compiled
package against any given stemcell line if the stemcell being used for a
deployment does not exactly match the version of the compiled package that is
referenced by a release. This means that if you have a package that is compiled
against Y.X and the same package compiled against Y.X+100, BOSH will prefer to
use the Y.X+100 over the Y.X package.

Thus if a later compiled package gets added to the director (Step 3), on any
subsequent deploys, BOSH may attempt to update usages of those packages in
deployments so that they are referencing the latest package (Step 4).
