# Updating BOSH Team Structure

## Context

The BOSH product is currently structured as two independently operating teams
that focus on different subsets of the BOSH product surface area. Currently they
are as follows:

* BOSH Director, 7 Engineers (4 San Francisco, 3 Toronto), 1 Product Managers:
  The BOSH Director team focuses mainly on the orchestration layer of BOSH known
  as the director. It's responsibilities include maintaining the bosh release
  for BOSH, the deployment repository, the CLI for interacting with the
  director, and various odds and ends (bosh.io, bosh-packages, AWS CPI, etc).
* BOSH Systems, 4 Engineers (4 San Francisco, 0 Toronto), 0 Product Managers:
  The BOSH Systems team focuses on building and releasing stemcells as well as
  building the BOSH DNS product for service discovery between BOSH deployed
  virtual machines.

These teams currently follow the typical Pivotal team processes including full
time pairing, IPM, retros, etc.

BOSH's product direction has recently been redefined according to strategy and
vision document[1].
The key take aways are that we would like BOSH to focus more on operational
stability and product maintainability. Any new features that will be added
should be for improving the PKS product or for reducing the support burden that
the BOSH support organization faces.

## Problem Definition

Due to some of the changes in priority around BOSH as a product, we have been
observing the following issues:

* The reactionary nature of feature requests and development has made it
  difficult for us to generate enough stories to satisfy a 5.5 pairs across two
  teams.
* BOSH Systems as a product space has proven to not be engaging enough for a
  full time product manager.
* Traditional Pivotal practices have made it difficult for engineers and product
  managers to find the time and space to appropriately engage future technologies
  such as Kubernetes.
* The deprioritization of BOSH product work can have a negative impact on
  engineers motivation and impression of their day to day efforts.

We also have noticed the following recurring issues on the BOSH teams as well:

* Engineer burn out due to an imbalance in context shared across the team.
* Difficulty ramping up and holding context in the platform automation space.
* Product uncertainty due to the technical nature of the product and the
  code/product debt that has been built up in the last few years.

## Proposal

In order to address the various issues listed above, we could deconstruct the
BOSH team into a non-standard Pivotal team that allows a set of core engineers
to maintain the product communally while allowing them the flexibility to focus
on other priorities for Pivotal as a company.

Specifically this would involve:

* Removing the separation between BOSH Director and BOSH Systems
* Scale the team down from 5.5 pairs to 6 independent engineers and 1 floating
  product manager with prior experience and passion in the BOSH ecosystem.
* Removing the expectation on full time pairing for engineers on a team. The
  engineers on the team should be able to decide when it is appropriate to pair.
* Removing the rigidity of the Pivotal process on weekly team interactions. This
  includes no IPM and optional retros when necessary.
* Product decisions are made communally and in an on-demand fashion, with input
  from a product manager.

The expectation would be that the engineers and product manager would spend a
good amount of time working on BOSH immediately after deconstructing the team (3
or 4 days a week).  The long term goal would be that the team discover ways to
drastically reduce the time necessary to maintain and develop for BOSH (2 days a
week). Members of the BOSH team would be expected to explore the Kubernetes
ecosystem and get involved with other teams in the platform program or other
related areas[2].

Some ideas for how this team could coordinate would be:

* Propose and discuss changes to improve BOSH in Github Issues and Pull
  Requests.
* Engineers generally submit pull requests to be reviewed by other team members
  in order to ensure that the team is sharing context and approving of changes.
* Team explores meeting structure that works for the level of collaboration that
  is necessary.
* All engineers are expected to be involved in the Pagerduty escalation
  schedule.
* Support issues during work hours will be handled asynchronously through slack
  channels and will be escalated through to the Pagerduty system when necessary.
* Define a schedule for team members to participate in the various external
  obligations that the BOSH team currently has, or work to cancel these
  obligations.

## Pros and Cons

This proposal is a drastic change from the typical Pivotal process and as a
result has a fair share of pros and cons associated with it. Some of the pros
could be:

* Allowing the team flexibility in their schedule to explore emerging
  technologies that will be the focus of Pivotal in the future.
* Increasing developer happiness for engineers that are looking for more self
  driven and independent formats.
* Reducing the amount of coordination that is necessary between team members
  that are across time zones.
* Allowing those with a passion for BOSH to explore features and improvements to
  reduce the support and maintainability of the product in the future.

Some of the cons that could exist are:

* Difficulty staffing the BOSH team due to a requirement of pre-existing BOSH
  context and interest.
* Difficulty sharing context amongst new members to the BOSH team.
* Independently driven BOSH product features may not have the same amount of
  product validation as they would with a full time BOSH product manager.
* Lack of accountability on BOSH team members to ensure that the product is
  being maintained acceptably.

## Measuring Success

In order to measure the success of this proposal we will need to have a regular
sync with leadership to ensure that we are satisfying all of the needs that the
organization has on the BOSH product. Some things to keep an eye on:

* Are all of the BOSH team members engaged with other Pivotal projects that are
  in the platform program? Different programs?
* Measure and monitor the number of support requests and issues. Ideally these
  should be decreasing over time.
* Engaging new engineers to explore the space as ways to sustainability staff
  the team going forward.
* Number of outstanding feature requests from existing products and PKS.

## Open Questions

* How do we staff the BOSH team? Open Call? Existing team members? Self
  selection?
  * Once the team is staffed, how do we handle rotations, exits, sustainability
    long term?
* What will the team's integration with BOSH Lifecycle be? Does the increase in
  flexibility allow us to work closely with the team to help solve their issues?
* Will rejoining the BOSH teams overwhelm members with the amount of scope
  involved? Or will high context engineers find the scope to be more manageable
  than a constantly rotating team?

## Links

[1] https://docs.google.com/document/d/1748X04ELyrcckHZm6GpOj4Worn5EqtwXQWESIS2_SOs
[2] https://docs.google.com/document/d/1LnaPZGi3hQKDzLiEhvKY8-BwxUiHlLlQsr143lqbVbs
