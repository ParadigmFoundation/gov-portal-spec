# Portal specification: `Governance`

The repository is home to the specification for the Kosu governance portal, primarily an interface for the `ValidatorRegistry` contract (a modified token-curated registry).

The repository contains the master `governance.sketch` file with the full UI/UX designs, and this README contains annotations of those designs and descriptions of each important state.

A custom helper library (`gov-portal-helper`) has been created to assist with the creation of this portal. Documentation for this library is found in the [documentation](#documentation) section.

# Background

Conceptually helpful topics and other background info required to work with this project. This document assumes familiarity with basic Ethereum concepts such as contracts, signatures, transactions, `web3.js`, providers, and MetaMask.

## Token curated registry

This governance portal is primarily an interface for a [token curated registry](https://medium.com/@ilovebagels/token-curated-registries-1-0-61a232f8dac7) (TCR), a type of Ethereum contract system designed to generate an arbitrary list of data by rewarding token-holders for curating its contents.

Paradigm's TCR curates a set of validators who run the Kosu network. Any entity may apply to the registry (create a "proposal"), where after a confirmation period they are accepted into the registry as a validator, so long as they have not been challenged. Applicants submit "stake" with the proposals, an amount of tokens they include with their proposal to indicate their seriousness of intent to join. At any point (even after the confirmation period) their listing may be "challenged", when another entity puts up an equal amount of stake to trigger a vote.

The resulting vote on the challenge ends either with the challenge passing, in which case the listing owner is removed from the registry and has their tokens distributed to the challenger and winning voters, or the challenge fails, in which case the listing remains and the challengers tokens are distributed to the listing owner and the winning voters.

## Vocabulary

- **Registry:** the TCR, specifically the Kosu `ValidatorRegistry` contract and the set of listings currently in the registry
- **Listing:** a non-specific term for any entity in the registry, either a _validator_ (confirmed) or a _proposal_ (pending, yet to be confirmed).
- **Proposal:** a new listing in the registry; a pending application to be a validator that is automatically confirmed after a period of time without being challenged.
- **Validator:** not only an actual validator on the proof-of-stake Kosu network, but more commonly in this context refers to a listing that has been confirmed to the registry.
- **Challenge:** an active poll against a listing (either a _proposal_ or a _challenge_) initiated by a token holder. A challenge resolves after a commit-reveal vote period, and results in distribution of the loser's (either the _challenger_ or the _listing_) stake to the winning voters and stakeholder (the _listing_ in the case of a failed challenge, or the _challenger_ in the case of a successful challenge).
- **Challenger:** an entity who has initiated a challenge against a _proposal_ or a _validator_ listing (note that a challenger who's challenge fails looses their staked tokens to the winning voters and the listing owner).
- **Voter:** an entity who is participating in a _challenge_ vote by committing their own tokens to one side of the poll (note, a voter may never loose their tokens if they vote on the losing side, they can only benefit if they win).
- **Stake:** a term used to describe A) an amount of tokens that may be lost in certain outcomes, or B) the act of committing tokens into a _staked_ position.
- **ValidatorRegistry:** the specific Ethereum contract that implements the functionality of this governance system within the greater Kosu protocol.

## Governance portal helper state/event model

The [`gov-portal-helper`](#documentation) library contains a single class, `Gov`, which manages and abstracts most functionality needed to interact with the Kosu system.

The three main properties listed below (and in more detail in the [documentation](#documentation) section) should be used for the main page (discussed below) that lists "proposals", "validators", and "challenges". These objects may be directly read from the `Gov` prototype, loaded via prototype methods, or queried each time a `gov_update` event is emitted from the `gov.ee` event-emitter object.

A `Gov` instance contains the following properties, which track the primary state of the portal system.

- [`gov.validators`](#Gov+currentValidators) - a map of Tendermint public keys (as hex-encoded strings) to [`Validator`](#validator) objects, with one entry for each current validator within the registry.
- [`gov.proposals`](#Gov+currentProposals) - a map of Tendermint public keys (as hex-encoded strings) to [`Proposal`](#proposal) objects, with one entry for each current pending proposal within the registry.
- [`gov.challenges`](#Gov+currentChallenges) - a map of Tendermint public keys (as hex-encoded strings) to [`StoreChallenge`](#StoreChallenge) objects, with one entry for listing that is currently being challenged.

All three properties above are loaded from the current contract system state when `gov.init()` is called, and further updated each time the contract system's state changes. After main page load, the `gov_update` event can be used to detect updates to the above state fields.

## Reliance on MetaMask

All sateful data (challenges, proposals, validators, past challenges, etc.) for the governance portal is loaded from the Ethereum blockchain, through the MetaMask extension which provides access to a remote Ethereum node.

As such, _this portal is useless without MetaMask,_ and MetaMask must be connected (via a call to `ethereum.enable()`) prior to any data being displayed. The actual call to `ethereum.enable()` is handled by initialization of a `Gov` instance (described below).

This brings up a few points to note:

1. Not having MetaMask installed should be treated as an error state, with a special page prompting the user to install the extension, or otherwise use a web3 browser.
1. On or just after page load, the `Gov` instance should be created. A call to [`gov.init()`](#govinit) handles the MetaMask connection, and prompts the user to grant site access. _The `gov.init()` call should not be made automatically,_ but instead should be triggered by the user clicking "connect to MetaMask" in the top nav bar.
1. Browser compatibility should also be handled. The only supported browsers are Chrome and Brave, unless a `window.ethereum` object is detected, in which case it can be assumed to be a compatible dApp browser.

_More details on `gov` initialization are included in the description and documentation sections._

## Units: wei vs. ether

Most token values (stake size, reward rate, etc.) are showed in "base units" of the token, which are also refereed to as `wei`, the smallest unit of most ERC-20 tokens. Becuase wei amounts will have between ~17-23 digits, amounts should be converted to ether prior to being displayed to the user.

A `gov` instance provides [`gov.weiToEther(wei)`](#govweitoetherwei--string) to convert from wei, and [`gov.etherToWei(ether)`](#govethertoweiether--string) to convert the other direction.

_These values are returned as strings, and should be converted to `BigNumber` instances where necessary. The static method `Gov.BigNumber` may be used to create new `BigNumber` instances from strings._

_Also note that "Ether" (proper noun) refers to the Ether cryptocurrency of the Ethereum network, while "ether" (lowercase) refers to the unit of 1\*10^18 wei._

# Description

This section contains the "meat" of the specification, with diagrams from [the sketch file](./governance.sketch) and annotations

## Main page (pre-MetaMask connection)

![Main page/overview state](./images/gov-load-state.png) <!-- https://sketch.cloud/s/VvZQ8/a/R4ZbmW -->

- This is the home/index page of the governance portal, which should be displayed prior to MetaMask connection
- The "Connect to MetaMask" button in the top-right nav-bar should trigger a call to [`gov.init()`](#govinit) which will prompt the user to allow the site access to their MetaMask `coinbase` account.
- Various exceptions (as promise rejections) from `gov.init()` indicate various failure states:
  ```javascript
  // init life-cycle (example only)
  (async () => {
    const gov = new Gov();
    try {
      await gov.init();
    } catch (error) {
      switch (error.message) {
        case "user denied site access":
          /* user clicked "reject" on MM prompt */
          break;
        case "non-ethereum browser detected":
          /* incompatible browser */
          break;
        default: {
          /* normal failure case, something unexpected went wrong */
        }
      }
    }
  })();
  ```

## Main page (post-MetaMask connection)

_The screenshots in this section are cropped from the same overview state, found [here](./images/gov-connected-main.png)._

![Main page/overview state (connected)](./images/gov-connected-nav-bar.png) <!-- https://sketch.cloud/s/VvZQ8/a/j0dowG -->

- After a successful (no exceptions) call to [`gov.init()`](#govinit), data for the main page can be loaded.
  - This call should be triggered by the user clicking "connect to MetaMask".
- The users balance of KOSU tokens should be displayed in the top-right nav par, next to their address.
  - The user's address is stored as `gov.coinbase` (after `init` has completed).
  - The user's Kosu balance can be loaded (as a `BigNumber`, units of wei) with the following code:
    ```javascript
    // the following requires `gov.init()` to have completed successfully
    const coinbase = gov.coinbase;
    const balance = await gov.kosu.kosuToken.balanceOf(coinbase);
    ```
- After initialization, the `gov` instance will begin to load validators, challenges, and proposals, into its state.
- Each time a new proposal, challenge, or validator is loaded, the `gov_update` event will be emitted from `gov.ee`.
- The raw map (object) for each of the following sub-sections can then be loaded from the `gov` instance.

### Active proposals

![Main page: active proposals](./images/gov-active-proposals.png) <!-- https://sketch.cloud/s/VvZQ8/a/j0dowG -->

- **Active proposals** should be loaded from `gov.proposals` (either by directly viewing that object, or a call to [`gov.currentProposals()`](#govcurrentproposals--mapproposal)).
  - Be sure to see the [`Proposal` type definition](#proposal) as well, which defines each object in the `gov.proposals` map.
  - Proposals on the main page display several values, which can be loaded (or computed) from each proposal object.
  - The titular hex string for each proposal card should be the key for the proposal in the `gov.proposals` object. This value is the public key for the listing application, encoded as a hex string.
  - The "new proposal" or "ending soon" badges should be displayed based on the `proposal.acceptUnix` field:
    - If `acceptUnix` is within one day of the current timestamp, "ending soon" should be displayed on the listing.
    - If `acceptUnix` is more than one week in the future, "new proposal" should be displayed on the listing.
    - If `acceptUnix` is less than a week in the future, but more than a day, no badge should be displayed.
  - The title for a proposal is "`X` wants to become a validator," and the `X` should be replaced with a shortened hex-string of the `proposal.owner` Ethereum address (`0x23ba...d2f`, for example).
    - Note that this hex string is 20 bytes (an Ethereum address, 42 chars with the "0x" prefix) and the listing key is a 32 byte (66 char with the "0x" prefix) Tendermint public key.
    - That is to say `proposal.owner` strictly is NOT EQUAL to the key of the object within the mapping.
  - The "Stake size" field should be loaded from `proposal.stakeSize`, and should be converted to units of ether prior to being displayed.
  - The "Daily reward" field should be loaded from `proposal.dailyReward` and should be converted to units of ether period to being displayed. Decimals can be truncated after 6-8 significant digits.
  - The "Estimated vote power" field should be loaded from `proposal.power`, which is a decimal number from 0-100 indicating a percentage of network vote power. The decimals can be truncated after a few significant digits, based on aesthetics/spacing.
  - The "Proposal ends in:" field must be computed based on the `proposal.acceptUnix` field, which indicates the Unix timestamp at which the proposal is confirmed. I recommend constructing a `Date` object from the timestamp, and using its methods to compute days, hours, and minutes. Seconds should not be shown, as the timestamp is purely an estimate.
  - The "View" button should take the user to the detail page for the proposal (detailed in a later section).

### Active challenges

![Main page: active challenges](./images/gov-active-challenges.png) <!-- https://sketch.cloud/s/VvZQ8/a/j0dowG -->

- **Active challenges** should be loaded from `gov.challenges` (either by directly viewing that object, or a call to [`gov.currentChallenges()`](#govcurrentchallenges--mapstorechallenge)).
  - Be sure to see the [`StoreChallenge` type definition](#storechallenge) as well, which defines each object in the `gov.challenges` map.
  - Challenges on the main page display a number, or index, which corresponds to the underlying `pollId` used to track that challenge in the contract system.
  - The number for each active challenge should be loaded from each `challenge.challengeId` and displayed (after conversion).
  - The "new challenge" or "ending soon" badge should be displayed on a challenge card, based on `challenge.challengeEndUnix`:
    - If `challengeEndUnix` is within a day of the current timestamp, "ending soon" should be displayed.
    - If `challengeEndUnix` is equal to or more than one week away, "new challenge" should be displayed.
    - If `challengeEndUnix` is in less than a week, but more than a day away, no badge should be displayed.
  - The "Challenger stake" field should be loaded from `challenge.challengerStake` and displayed after conversion to units of ether.
  - The "Potential reward" field must be computed based on the value of `challenge.challengerStake`:
    - The maximum possible reward refers to the proportion of listing/challenger stake that could be rewarded to participating voters.
    - This value is based on a variable contract parameter, but for now, can be _assumed to be 30% of the `challengerStake`_.
    - Keep in mind `challenge.challengerStake` is an instance of `BigNumber`, so the appropriate methods must be used to calculate `0.3 * challengerStake`.
  - The "Challenge ends in:" field should be computed, much like the "Proposal ends in:" field above, from each `challenge.challengeEndUnix` value which is the estimated Unix timestamp (in seconds) that the challenge will end.
  - The displayed countdown should only be precise to the minute, as the provided timestamp is simply an estimation.
  - The "View" button on each card should bring the user to the detail page for that challenge (discussed in a further section).

### Validators

![Main page: validators table](./images/gov-validators-table.png) <!-- https://sketch.cloud/s/VvZQ8/a/j0dowG -->

- **Validators** should be loaded from `gov.validators` (either by directly viewing that object, or a call to [`gov.currentValidators()`](#govcurrentvalidators--mapvalidator)).
  - Be sure to see the [`Validator` type definition](#validator) as well, which defines each object in the `gov.validators` map.
  - The validators table has the following column headers:
    - **Address** is the Ethereum address of a given validator.
    - **Stake** is the validator Kosu token stake size, displayed in ether units.
    - **Daily reward** is the estimated number of tokens minted to the validator per day (in units of ether).
    - **Vote power** is a percentage indicating a given validators influence and power during consensus.
    - **Uptime** is a percentage representing how many blocks the validator has signed compared to how many they should have signed, if they were online all the time. This value for now should be populated as "N/A" for each listing. Eventually, this value will be loaded from a remote RPC API.
    - **Age** is simply how long a given validator has been confirmed in the registry (displayed in human units of time, even though the value is represented as a block number).
  - For each validator (each `validator` in `gov.validators`), the fields described above should be loaded/computed as follows.
    - **Address** should be the Ethereum address loaded from `validator.owner`, and can be displayed as a "hyphenated" hex string.
    - **Stake** should be loaded from `validator.stakeSize` (which is a `BigNumber`) and converted to units of ether prior to being displayed.
    - **Daily reward** should be loaded from `validator.dailyReward`, a `BigNumber` instance, and converted to units of ether prior to being displayed.
    - **Vote power** should be loaded from `validator.power` which is a `BigNumber` instance with a value between 0 and 100. The decimal value should be truncated so as to not display too many decimals. This truncation can be done based on space, or a fixed number of significant digits. Keep in mind, a validator may have less than 1% vote power.
    - **Uptime** is not a real value at this time, and should be displayed as "N/A" for each listing.
    - **Age** should be computed based on `validator.confirmationUnix` which is the Unix timestamp in seconds that the validator was confirmed. The "Age" value should display the number of days, hours, and minutes since this time.
  - Clicking on a validator entry (one of the rows) should take the user to the validator detail page for that listing (described in a later section).

### Past challenges

![Main page: validators table](./images/gov-past-challenges-table.png) <!-- https://sketch.cloud/s/VvZQ8/a/j0dowG -->

- **Past challenges** is a section containing a table with information about past challenges and the results of each.
  - Be sure to be familiar with the [`PastChallenge`](#PastChallenge) and [`ListingSnapshot`](#ListingSnapshot) types.
  - Past challenges are stored and loaded separately from the primary `gov` properites (`proposals`, `challenges`, and `validators`) and are not updated in real time.
    - Instead, past challenges must be loaded by calling [`gov.getHistoricalChallenges()`](#govgethistoricalchallenges--promisearraypastchallenge):
      ```javascript
      // following requires `gov.init()` to have completed successfully
      const pastChallenges = await gov.getHistoricalChallenges();

      // the number of challenges (use index +1 to display under "ID")
      console.log(pastChallenges.length);
      ```
- The "Past Challenges" table has the following column headers:
  - **ID** is the challenge's unique ID, which increments from 0.
  - **Challenger** is the Ethereum address of the entity that initiated the challenge.
  - **Type** indicates weather the challenge was against an active "validator" or against a pending "proposal".
  - **Result** indicates the result of the challenge, where "accepted" indicates the challenge won, and "rejected" indicates the challenge failed/lost.
  - **Tokens at Stake** is the total number of tokens at stake during the challenge, equal to the challenge stake plus the listing owner's stake (see below).
  - **Time** indicates the time a given challenge ended (vote period ends).
- For each table entry (loaded from the array returned by [`gov.getHistoricalChallenges()`](#govgethistoricalchallenges-⇒-promisearraypastchallenge)) and each column field above:
  - **ID** can be loaded from `challenge.listingSnapshot.currentChallenge` or can be loaded from the challenge's index within the array returned by `getHistoricalChallenges`.
  - **Challenger** should be loaded from `challenge.challenger` (an Ethereum address).
  - **Type** is computed based on `challenge.listingSnapshot.status` based on the following:
    - If `listingSnapshot.status === 1`, "Type" should be "Proposal" (blue).
    - If `listingSnapshot.status === 2`, "Type" should be "Validator" (orange).
    - If `listingSnapshot.status` is **not** `1` or `2`, _that entry should not be displayed._
  - **Result** should be based off the boolean `challenge.passed` field:
    - If `challenge.passed === true` then "Result" should be "Accepted" (green).
    - If `challenge.passed === false` then "Result" should be "Rejected" (red).
  - **Tokens at Stake** should be computed as the sum of `challenge.balance` and `challenge.listingSnapshot.stakedBalance` and displayed in units of ether.
    - Keep in mind both `challenge.balance` and `challenge.listingSnapshot.stakedBalance` are `BigNumber` instances, and stored in units of wei which must be converted prior to displaying.
  - **Time** should be calculated based on the timestamp of the `challenge.challengeEnd` block number.
    - This timestamp can be loaded from the [`gov.getPastBlockTimestamp(n)`](#govgetpastblocktimestampblocknumber--promisenumber) method, where `n` is passed in as `challenge.challengeEnd`.
    - For example:
      ```javascript
      // gov.init() must have completed successfully prior to this working
      for (let i = 0; i < challenges.length; i++) {
        const challenge = challenges[i];
        const challengeEndBlock = challenge.challengeEnd.toNumber();

        // use this value to display "Time"
        const challengeEndTimestamp = await gov.getPastBlockTimestamp(
          challengeEndBlock
        );
      }
      ```
- Clicking on a challenge should take to a past challenge detail page for that challenge (discussed in a later section).

## Proposal detail page
- Clicking on a proposal from the main page directs the user to the proposal detail page (below).
- On the proposal detail page, the user can be prompted to challenge the proposal.
- In addition to the screenshots below, be sure to see [the mobile version](./images/gov-proposal-page-mobile.png), and the full sketch file.

### Main detail

![Proposal detail page: main](./images/gov-proposal-page.png) <!-- https://sketch.cloud/s/VvZQ8/a/wpmj8p -->

- This is the primary detail page for active proposals.
- The majority of the data for this page can be loaded from the corresponding `gov.proposals` object.
- Each detail page should correspond to one of the objects in `gov.proposals`, where each `proposal` is key'd by the listing's Tendermint public key.
- Page features/fields:
  - **Tendermint public key** (mid/top left) should correspond to the object property key of the current `gov.proposals` object.
  - **`X` wants to become a validator** the value for `X` (Ethereum address) should be loaded from `proposal.owner`.
  - **If unchallenged...** the countdown to when the proposal becomes a validator should be computed based on the `proposal.acceptUnix` and should be displayed as a countdown of days, hours, and minutes (seconds should be ignored due to `acceptUnix` being an estimate).
  - Card: **Stake size** should be loaded from `proposal.stakeSize` (remember it is a `BigNumber`, and must be converted to ether prior to display).
  - Card: **Daily reward** should be loaded from `proposal.dailyReward` (must be converted to ether prior to being displayed).
  - Card: **Estimated vote power** should be loaded from `proposal.power` and displayed as a percentage, keeping in mind some digits should not be displayed (i.e. trailing decimals).
  - Button: **Challenge proposal** triggers a change to the [next state](#challenge-prompt) and can ultimately lead to a call to initiate a challenge (see next sub-section).

### Challenge prompt

![Proposal detail page: challenge](./images/gov-proposal-challenge.png) <!-- https://sketch.cloud/s/VvZQ8/a/wpmj8p -->

- This state offers the user another opportunity to confirm their intent to challenge a listing.
- It provides a text entry field for the user to provided additional information to support their challenge.
- The "this challenge will cost" field should be loaded from the value of `proposal.stakeSize` and displayed in units of ether.
- The "Challenge" button should trigger the following (async code):
  ```javascript
  // inside async method, assume `proposal` is the proposal object from gov.proposals
  // assume `listingPubKey` is the key of that `proposal` object within the mapping
  const stringDetails = loadStringDetails() // pseudocode, load from input box

  // the current proposal's public key
  const listingKey = listingPubKey;

  const receipt = await gov.kosu.validatorRegistry.challengeListing(
    listingKey,
    stringDetails,
  );

  // `receipt.transactionHash` is the txId of the challenge (can be used to view on Etherscan, etc).
  ```

## Active challenge detail

- These states exist for active challenges (currently in `gov.challenges`).
- The first state below (["main detail"](#main-detail-commit-period)) shows information about the challenge, and allows the user to vote (see subsequent sub-sections).
- The "Vote on this challenge" section will vary based on what phase the challenge is in:
  - **Commit period** starts immediately after the challenge is initiated, and it lasts until the reveal period starts. 
  - **Reveal period** starts after `commitPeriod` blocks from the challenge initialization, and lasts until the `challengeEnd` block. After that point, the challenge is finalized and should be accessed using the `getHistoricalChallenges` method. 
- Be sure to view the full sketch file for un-included states that can be inferred from others.
- There are additional screenshots in [`./images`](./images) for various assumed states, as well as mobile layout.

### Note: detecting commit/reveal period

During the commit period, the first state below should be shown (under "main detail"), and during the reveal period, the "reveal period" state should be shown, where the two voting buttons are replaced with a single "reveal vote" button.

Example of detecting commit vs reveal period for challenge **#5**:
```javascript
// gov.init() must have been called at this point. 
const currentBlock = await gov.currentBlockNumber();

const challengeInfo = await gov.getChallengeInfo("5");
const {
  challengeStart,
  endCommitPeriod,
  challengeEnd,
} = challengeInfo;

if (challengeStart <= currentBlock && currentBlock < endCommitPeriod) {
  // challenge is _IN COMMIT PERIOD_
  // votes _MAY BE COMMITTED_
} else if (endCommitPeriod <= currentBlock && currentBlock < challengeEnd) {
  // challenge is _IN REVEAL PERIOD_
  // votes _MAY NOT BE COMMITTED, ONLY REVEALED_
} else if (challengeEnd <= currentBlock) {
  // reveal period is OVER, and challenge is finalized
}
```

### Main detail (commit period)

_This state shows a challenge during the commit phase._

![Challenge detail page: main](./images/gov-active-challenge-page.png) <!-- https://sketch.cloud/s/VvZQ8/a/4GeWPm -->

- The detail page exists for each `challenge` in the `gov.challenges` object.
- This state should be shown when `challenge.challengeType === "validator"`.
- Page fields/values:
  - **Challenge #`N`** where `N` is the `challenge.challengeId` value (as a string).
  - **Validator's public key** is the object key (public key) in `gov.challenges` that corresponds to this `challenge` object.
  - **`X` is challenging validator `Y`** where `X` is `challenge.challenger` and `Y` is `challenge.listingOwner`.
    - Note, this state shows a validator challenge. If the challenge is against a proposal (`challenge.challengeType === "proposal") a separate state should be shown. 
  - **Entity y believes that...** this string should be loaded and populated from `challenge.challengeDetails`.
  - **This challenge will end in...** this countdown should be computed based on `challenge.challengeEndUnix` and should update in real-time (seconds should now be shown).
  - **Potential reward** should be equal to 30% of `challenge.challengerStake` (display in ether).
  - **Challenger stake** should be loaded from `challenge.challengerStake` (and displayed in ether).
  - **Validator voting power** should only be displayed if `challenge.challengeType === "validator"` and can be computed as follows: 
    ```javascript
    // assume `listingPublicKey` is the challenge-ee public key (the key in `gov.challenges`)
    const powerBigNum = await gov._getValidatorPower(listingPublicKey); // BigNumber
    ```
- The voting section should be displayed on both "validator" and "proposal" challenges.
- Depending on the [challenge state](#note-detecting-commitreveal-period) the buttons should display either:
  - Two buttons (as seen in image), one for each option if in the commit period.
  - One "reveal" button (displayed in later section) if the challenge is in the reveal period.

### Main detail (proposal, commit period)

![Challenge detail page: proposal](./images/gov-active-challenge-proposal.png) <!-- https://sketch.cloud/s/VvZQ8/a/Zy9wv7 -->

- If `challenge.challengeType === "proposal"` this state should be used (minor changes from above).
- The commit/reveal logic is the same for when to display voting buttons vs. when to display "reveal" button (see [here](#note-detecting-commitreveal-period)).

### Confirm vote

![Challenge detail page: confirm commit vote](./images/gov-challenge-confirm-vote.png)

- During the commit period, users may vote to remove or vote to keep (both challenges and proposals).
- Clicking either vote option prompts for another confirmation.
- Subsequent confirmation should trigger the following logic:
  ```javascript
  const tokensToCommit; // load from user (keep in mind wei conversion), should be a `BigNumber`
  const challengeId;    // load from challenge page, should be `BigNumber`

  /**
   * Vote "value" is a string, either "1" or "0":
   * - use "1" if the vote is to support the challenge (remove a validator/proposal)
   * - use "0" if the vote is to support the validator/proposal (vote against the challenge)
   */
  const value = "1"

  // will prompt user to sign transaction
  // will fail if user does not have enough tokens or allowance
  // this will store the vote salt and value in a cookie, to be used later for reveal
  const txId = await gov.commitVote(challengeId, value, tokensToCommit);
  ```

### After commit vote

![Challenge detail page: post-commit](./images/gov-challenge-post-vote.png)

- This state should be shown after a successful commit vote transaction (above).
- For now, "account page" should be a null link (`#`).
- The "Add a reminder" link should use the Google Calendar URL API to create an event (see [example](https://github.com/decomaan/google-calendar-link-generator/blob/master/app/js/app.js)).
- The event can be created for the duration of the "reveal period" which can be calculated as follows:
  ```javascript
  const challengeStart = challenge.challengeEnd - gov.params.challengePeriod;
  const startReveal = challengeStart + gov.params.commitPeriod;
  const endReveal = challenge.challengeEnd;

  // the calendar event should be created between these two unix timestamps
  const startRevealUnix = await gov.estimateFutureBlockTimestamp(startReveal)
  const endRevealUnix = await gov.estimateFutureBlockTimestamp(endReveal);
  ```

### Reveal vote

![Challenge detail page: reveal period](./images/gov-challenge-page-reveal.png)

- This state should be shown when a challenge is in the "reveal" period.
- Weather a challenge is in reveal period or not can be determined [as described here](#note-detecting-commitreveal-period).
- If the user clicks "reveal", the following logic should be triggered:
  ```javascript
  const challengeId = new BigNumber(challenge.challengeId);

  // will trigger MetaMask popup, assuming a vote was previously committed from the same browser
  const revealTxId = await gov.revealVote(challengeId);
  ```
_Be sure to catch promise rejections from `gov.revealVote` in the case that the user did not commit a vote from the same browser._
# Documentation

- Below is the README for `@kosu/gov-portal-helper` package.
- The published package may be found [on the NPM registry](https://www.npmjs.com/package/@kosu/gov-portal-helper).
- For safety from `number` precision issues, most numerical values are expected and returned as [`BigNumber`](https://github.com/MikeMcl/bignumber.js) instances, so be sure to be familiar with [that API](http://mikemcl.github.io/bignumber.js/).

## `@kosu/gov-portal-helper`

<p><code>Gov</code> is a helper library for interacting with the Kosu validator governance
system (primarily the Kosu <code>ValidatorRegistry</code> contract).</p>
<p>It is designed with the browser in mind, and is intended to be used in front-
end projects for simplifying interaction with the governance system.</p>

## Installation

Add `gov-portal-helper` to your project via `npm` or `yarn`.

```shell
# install with yarn
yarn add @kosu/gov-portal-helper

# install with npm
yarn add @kosu/gov-portal-helper
```

## Typedefs

<dl>
<dt><a href="#Validator">Validator</a></dt>
<dd><p>Represents an active validator in the registry.</p></dd>
<dt><a href="#Proposal">Proposal</a></dt>
<dd><p>Represents a pending listing application for a spot on the <code>ValidatorRegistry</code>.</p></dd>
<dt><a href="#StoreChallenge">StoreChallenge</a></dt>
<dd><p>Represents a current challenge (in the <code>gov</code> state).</p></dd>
<dt><a href="#PastChallenge">PastChallenge</a></dt>
<dd><p>Represents a historical challenge, and its outcome.</p></dd>
<dt><a href="#ListingSnapshot">ListingSnapshot</a></dt>
<dd><p>Represents a listing at the time it was challenged.</p></dd>
<dt><a href="#Vote">Vote</a></dt>
<dd><p>Represents a stored vote in a challenge poll.</p></dd>
</dl>

<a name="Gov"></a>

## Gov
<p><code>Gov</code> is a helper library for interacting with the Kosu validator governance
system (primarily the Kosu <code>ValidatorRegistry</code> contract).</p>
<p>It is designed with the browser in mind, and is intended to be used in front-
end projects for simplifying interaction with the governance system.</p>
<p>Methods may be used to load the current <code>proposals</code>, <code>validators</code>, and
<code>challenges</code> from the prototype's state, or the <code>gov.ee</code> object (an EventEmitter)
may be used to detect updates to the state, emitted as <code>gov_update</code> events.</p>
<p>After a <code>gov_update</code> event, the read methods for <code>proposals</code>, <code>challenges</code>,
and <code>validators</code> must be called to load the current listings. Alternatively,
access the objects directly with <code>gov.listings</code>, etc.</p>

**Kind**: global class  

* [Gov](#Gov)
    * [new Gov()](#new_Gov_new)
    * _instance_
        * [.init()](#Gov+init)
        * [.currentProposals()](#Gov+currentProposals) ⇒ [<code>Map.&lt;Proposal&gt;</code>](#Proposal)
        * [.currentValidators()](#Gov+currentValidators) ⇒ [<code>Map.&lt;Validator&gt;</code>](#Validator)
        * [.currentChallenges()](#Gov+currentChallenges) ⇒ [<code>Map.&lt;StoreChallenge&gt;</code>](#StoreChallenge)
        * [.weiToEther(wei)](#Gov+weiToEther) ⇒ <code>string</code>
        * [.etherToWei(ether)](#Gov+etherToWei) ⇒ <code>string</code>
        * [.commitVote(challengeId, value, amount)](#Gov+commitVote) ⇒ <code>Promise.&lt;string&gt;</code>
        * [.revealVote(challengeId)](#Gov+revealVote) ⇒ <code>Promise.&lt;string&gt;</code>
        * [.estimateFutureBlockTimestamp(blockNumber)](#Gov+estimateFutureBlockTimestamp) ⇒ <code>Promise.&lt;number&gt;</code>
        * [.getPastBlockTimestamp(blockNumber)](#Gov+getPastBlockTimestamp) ⇒ <code>Promise.&lt;number&gt;</code>
        * [.getHistoricalChallenges()](#Gov+getHistoricalChallenges) ⇒ <code>Promise.&lt;Array.&lt;PastChallenge&gt;&gt;</code>
        * [.getChallengeInfo(challengeId)](#Gov+getChallengeInfo) ⇒ <code>Promise.&lt;ChallengeInfo&gt;</code>
        * [.currentBlockNumber()](#Gov+currentBlockNumber) ⇒ <code>number</code>
    * _static_
        * [.ZERO](#Gov.ZERO)
        * [.ONE](#Gov.ONE)
        * [.ONE_HUNDRED](#Gov.ONE_HUNDRED)
        * [.BLOCKS_PER_DAY](#Gov.BLOCKS_PER_DAY)

<a name="new_Gov_new"></a>

### new Gov()
<p>Create a new <code>Gov</code> instance (<code>gov</code>). Requires no arguments, but can be
set to &quot;debug&quot; mode by passing <code>true</code> or <code>1</code> (or another truthy object to
the constructor).</p>
<p>Prior to using most <code>gov</code> functionality, the async <code>gov.init()</code> method
must be called, which will initialize the module and load state from
the Kosu contract system.</p>

<a name="Gov+init"></a>

### gov.init()
<p>Main initialization function for the <code>gov</code> module. You must call <code>init</code>
prior to interacting with most module functionality, and <code>gov.init()</code> will
load the current registry status (validators, proposals, etc.) so it should
be called early-on in the page life-cycle.</p>
<p>Performs many functions, including:</p>
<ul>
<li>prompt user to connect MetaMask</li>
<li>load user's address (the &quot;coinbase&quot;)</li>
<li>load the current Ethereum block height</li>
<li>load and process the latest ValidatorRegistry state</li>
</ul>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
<a name="Gov+currentProposals"></a>

### gov.currentProposals() ⇒ [<code>Map.&lt;Proposal&gt;</code>](#Proposal)
<p>Load the current proposal map from state.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: [<code>Map.&lt;Proposal&gt;</code>](#Proposal) - <p>a map where the key is the listing public key, and the value is a proposal object</p>  
**Example**  
```javascript
const proposals = gov.currentProposals();
```
<a name="Gov+currentValidators"></a>

### gov.currentValidators() ⇒ [<code>Map.&lt;Validator&gt;</code>](#Validator)
<p>Load the current validators map from state.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: [<code>Map.&lt;Validator&gt;</code>](#Validator) - <p>a map where the key is the listing public key, and the value is a validator object</p>  
**Example**  
```javascript
const validators = gov.currentValidators();
```
<a name="Gov+currentChallenges"></a>

### gov.currentChallenges() ⇒ [<code>Map.&lt;StoreChallenge&gt;</code>](#StoreChallenge)
<p>Load the current challenges map from state.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: [<code>Map.&lt;StoreChallenge&gt;</code>](#StoreChallenge) - <p>a map where the key is the listing public key, and the value is a challenge object</p>  
**Example**  
```javascript
const challenges = gov.currentChallenges();
```
<a name="Gov+weiToEther"></a>

### gov.weiToEther(wei) ⇒ <code>string</code>
<p>Convert a number of tokens, denominated in the smallest unit - &quot;wei&quot; - to
&quot;full&quot; units, called &quot;ether&quot;. One ether = 1*10^18 wei.</p>
<p>All contract calls require amounts in wei, but the user should be shown
amounts in ether.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>string</code> - <p>the same amount in ether, string used for precision</p>  

| Param | Type | Description |
| --- | --- | --- |
| wei | <code>BigNumber</code> \| <code>string</code> | <p>the token amount in wei to convert</p> |

**Example**  
```javascript
gov.weiToEther("100000000000000000000") // > "100"
gov.weiToEther(100000000000000000000)   // > "100"
```
<a name="Gov+etherToWei"></a>

### gov.etherToWei(ether) ⇒ <code>string</code>
<p>Convert a number of tokens (full units, called &quot;ether&quot;) to &quot;wei&quot;, the
smallest denomination of most ERC-20 tokens with 18 decimals.</p>
<p>All contract calls require amounts in wei, but the user should be shown
amounts in ether.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>string</code> - <p>the same amount in wei, string used for precision</p>  

| Param | Type | Description |
| --- | --- | --- |
| ether | <code>BigNumber</code> \| <code>string</code> | <p>the token amount to convert</p> |

**Example**  
```javascript
gov.etherToWei(10)  // > "10000000000000000000"
gov.etherToWei("1") // > "1000000000000000000"
```
<a name="Gov+commitVote"></a>

### gov.commitVote(challengeId, value, amount) ⇒ <code>Promise.&lt;string&gt;</code>
<p>Commit a vote in an active a challenge poll.</p>
<p>This method creates a vote (with value and salt), encodes it, and submits
it as a transaction (requires MetaMask signature).</p>
<p>It stores the vote data in a browser cookie so it may be revealed later,
which means voters must reveal a vote with the same browser they used to
commit it.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;string&gt;</code> - <p>the transaction hash of the commit tx</p>  

| Param | Type | Description |
| --- | --- | --- |
| challengeId | <code>BigNumber</code> | <p>the pollId of the challenge, as a string</p> |
| value | <code>string</code> | <p>the vote value, &quot;1&quot; to vote for the challenge, &quot;0&quot; to vote against</p> |
| amount | <code>BigNumber</code> | <p>the number of tokens (in wei) to commit in the vote</p> |

**Example**  
```javascript
// we are looking at challenge #13, and want to vote AGAINST it with 10 tokens
const pollId = new BigNumber(13);
const amount = new BigNumber(gov.etherToWei("10"));
const value = "0";

// will prompt for MetaMask signature
const commitVoteTxId = await gov.commitVote(pollId, value, amount);

// ... some time passes, we now want to reveal ...

// load vote from cookie and reveal
const revealTxId = await gov.revealVote(new BigNumber("13"));

// ... wait for Tx's to confirm or whatever, etc.
```
<a name="Gov+revealVote"></a>

### gov.revealVote(challengeId) ⇒ <code>Promise.&lt;string&gt;</code>
<p>Reveal a previously committed vote, by challengeId (as a BigNumber).</p>
<p>For this method to work, the user must have committed a vote during the
commit period for the given challenge.</p>
<p>This method must also be called during the reveal period in order for the
transaction not to fail.</p>
<p>Calling this method will trigger a MetaMask pop-up asking for the user's
signature and approval.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;string&gt;</code> - <p>the transaction hash of the reveal tx.</p>  

| Param | Type | Description |
| --- | --- | --- |
| challengeId | <code>BigNumber</code> | <p>the challenge to reveal a stored vote for</p> |

<a name="Gov+estimateFutureBlockTimestamp"></a>

### gov.estimateFutureBlockTimestamp(blockNumber) ⇒ <code>Promise.&lt;number&gt;</code>
<p>Estimate the UNIX timestamp (in seconds) at which a given <code>block</code> will be
mined.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;number&gt;</code> - <p>the block's estimated UNIX timestamp (in seconds)</p>  

| Param | Type | Description |
| --- | --- | --- |
| blockNumber | <code>number</code> | <p>the block number to estimate the timestamp for</p> |

**Example**  
```javascript
const block = 6102105;
const unixTs = gov.estimateFutureBlockTimestamp(block);

// use as a normal date object (multiply by 1000 to get ms)
const blockDate = new Date(ts * 1000);
```
<a name="Gov+getPastBlockTimestamp"></a>

### gov.getPastBlockTimestamp(blockNumber) ⇒ <code>Promise.&lt;number&gt;</code>
<p>Retrieve the Unix timestamp of a block that has already been mined.
Should be used to display times of things that have happened (validator
confirmed, etc.).</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;number&gt;</code> - <p>the Unix timestamp of the specified <code>blockNumber</code></p>  

| Param | Type | Description |
| --- | --- | --- |
| blockNumber | <code>number</code> | <p>the block to get the unix timestamp for</p> |

**Example**  
```javascript
await gov.getPastBlockTimestamp(515237) // > 1559346404
```
<a name="Gov+getHistoricalChallenges"></a>

### gov.getHistoricalChallenges() ⇒ <code>Promise.&lt;Array.&lt;PastChallenge&gt;&gt;</code>
<p>This method returns an array (described below) that contains information
about all past challenges. Intended to be used for the &quot;Past Challenges&quot;
section.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;Array.&lt;PastChallenge&gt;&gt;</code> - <p>all historical <code>challenges</code>.</p>  
<a name="Gov+getChallengeInfo"></a>

### gov.getChallengeInfo(challengeId) ⇒ <code>Promise.&lt;ChallengeInfo&gt;</code>
<p>Returns an object with the block numbers of important times for a given
challenge. Between <code>challengeStart</code> and <code>endCommitPeriod</code>, votes may be
committed (submitted) to the challenge.</p>
<p>Between <code>endCommitPeriod</code> and <code>challengeEnd</code>, votes may be revealed with
the same salt and vote value.</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>Promise.&lt;ChallengeInfo&gt;</code> - <p>the block numbers for this challenge</p>  

| Param | Type | Description |
| --- | --- | --- |
| challengeId | <code>string</code> \| <code>number</code> \| <code>BigNumber</code> | <p>the ID of the challenge to query</p> |

**Example**  
```javascript
const info = await gov.getChallengeInfo(new BigNumber(1));
const currentBlock = await gov.currentBlockNumber();

if (currentBlock < endCommitPeriod && currentBlock >= challengeStart) {
  // in "commit" period; voters may submit votes
} else if (currentBlock >= endCommitPeriod && currentBlock <= challengeEnd) {
  // in "reveal" period; voters may reveal votes
} else {
  // challenge has ended (or issues with block numbers)
}
```
<a name="Gov+currentBlockNumber"></a>

### gov.currentBlockNumber() ⇒ <code>number</code>
<p>Returns the current block height (as a number).</p>

**Kind**: instance method of [<code>Gov</code>](#Gov)  
**Returns**: <code>number</code> - <p>The current (or most recent) Ethereum block height.</p>  
<a name="Gov.ZERO"></a>

### Gov.ZERO
<p>The value <code>0</code> as an instance of<code>BigNumber</code>.</p>

**Kind**: static property of [<code>Gov</code>](#Gov)  
<a name="Gov.ONE"></a>

### Gov.ONE
<p>The value <code>1</code> as an instance of<code>BigNumber</code>.</p>

**Kind**: static property of [<code>Gov</code>](#Gov)  
<a name="Gov.ONE_HUNDRED"></a>

### Gov.ONE\_HUNDRED
<p>The value <code>100</code> as an instance of<code>BigNumber</code>.</p>

**Kind**: static property of [<code>Gov</code>](#Gov)  
<a name="Gov.BLOCKS_PER_DAY"></a>

### Gov.BLOCKS\_PER\_DAY
<p>Estimated blocks per day (mainnet only).</p>

**Kind**: static property of [<code>Gov</code>](#Gov)  
<a name="Validator"></a>

## Validator
<p>Represents an active validator in the registry.</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| owner | <code>string</code> | <p>the Ethereum address of the validator</p> |
| stakeSize | <code>BigNumber</code> | <p>the staked balance (in wei) of the validator</p> |
| dailyReward | <code>BigNumber</code> | <p>the approximate daily reward to the validator (in wei)</p> |
| confirmationUnix | <code>number</code> | <p>the unix timestamp of the block the validator was confirmed in</p> |
| power | <code>BigNumber</code> | <p>the validators approximate current vote power on the Kosu network</p> |
| details | <code>string</code> | <p>arbitrary details provided by the validator when they applied</p> |

<a name="Proposal"></a>

## Proposal
<p>Represents a pending listing application for a spot on the <code>ValidatorRegistry</code>.</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| owner | <code>string</code> | <p>the Ethereum address of the applicant</p> |
| stakeSize | <code>BigNumber</code> | <p>the total stake the applicant is including with their proposal (in wei)</p> |
| dailyReward | <code>BigNumber</code> | <p>the approximate daily reward (in wei) the applicant is requesting</p> |
| power | <code>BigNumber</code> | <p>the estimated vote power the listing would receive if accepted right now</p> |
| details | <code>string</code> | <p>arbitrary details provided by the applicant with their proposal</p> |
| acceptUnix | <code>number</code> | <p>the approximate unix timestamp the listing will be accepted, if not challenged</p> |

<a name="StoreChallenge"></a>

## StoreChallenge
<p>Represents a current challenge (in the <code>gov</code> state).</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| listingOwner | <code>string</code> | <p>the Ethereum address of the owner of the challenged listing</p> |
| listingStake | <code>BigNumber</code> | <p>the total stake of the challenged listing</p> |
| listingPower | <code>BigNumber</code> | <p>the current vote power of the listing (if they are a validator)</p> |
| challenger | <code>string</code> | <p>the Ethereum address of the challenger</p> |
| challengeId | <code>BigNumber</code> | <p>the incremental ID of the current challenge</p> |
| challengerStake | <code>BigNumber</code> | <p>the staked balance of the challenger</p> |
| challengeEndUnix | <code>number</code> | <p>the estimated unix timestamp the challenge ends at</p> |
| challengeEnd | <code>BigNumber</code> | <p>the block at which the challenge reveal period ends</p> |
| totalTokens | <code>BigNumber</code> | <p>if finalized, the total number of tokens from participating voters</p> |
| winningTokens | <code>BigNumber</code> | <p>if finalized, the number of tokens that voted on the winning side</p> |
| result | <code>string</code> | <p>the final result of the challenge; &quot;passed&quot;, &quot;failed&quot;, or <code>null</code> if not finalized</p> |
| challengeType | <code>string</code> | <p>the type of listing the challenge is against, either a &quot;validator&quot; or a &quot;proposal&quot;</p> |
| listingDetails | <code>string</code> | <p>details provided by the listing holder</p> |
| challengeDetails | <code>string</code> | <p>details provided by the challenger</p> |

<a name="PastChallenge"></a>

## PastChallenge
<p>Represents a historical challenge, and its outcome.</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| balance | <code>BigNumber</code> | <p>the number of tokens (in wei) staked in the challenge</p> |
| challengeEnd | <code>BigNumber</code> | <p>the block the challenge ends at</p> |
| challenger | <code>string</code> | <p>the Ethereum address of the challenger</p> |
| details | <code>string</code> | <p>additional details provided by the challenger</p> |
| finalized | <code>boolean</code> | <p><code>true</code> if the challenge result is final, <code>false</code> if it is ongoing</p> |
| listingKey | <code>string</code> | <p>the key that corresponds to the challenged listing</p> |
| listingSnapshot | [<code>ListingSnapshot</code>](#ListingSnapshot) | <p>an object representing the state of the challenged listing at the time of challenge</p> |
| passed | <code>boolean</code> | <p><code>true</code> if the challenge was successful, <code>false</code> otherwise</p> |
| pollId | <code>BigNumber</code> | <p>the incremental ID used to identify the poll</p> |
| voterTotal | <code>BigNumber</code> | <p>the total number of tokens participating in the vote</p> |
| winningTokens | <code>BigNumber</code> | <p>the total number of tokens voting for the winning option</p> |

<a name="ListingSnapshot"></a>

## ListingSnapshot
<p>Represents a listing at the time it was challenged.</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| applicationBlock | <code>BigNumber</code> | <p>the block the listing application was submitted</p> |
| confirmationBlock | <code>BigNumber</code> | <p>the block the listing was confirmed (0 if unconfirmed)</p> |
| currentChallenge | <code>BigNumber</code> | <p>the ID of the current challenge against the listing</p> |
| details | <code>string</code> | <p>arbitrary details provided by the listing applicant</p> |
| exitBlock | <code>BigNumber</code> | <p>the block (if any) the listing exited at</p> |
| lastRewardBlock | <code>BigNumber</code> | <p>the last block the listing owner claimed rewards for</p> |
| owner | <code>string</code> | <p>the Ethereum address of the listing owner</p> |
| rewardRate | <code>BigNumber</code> | <p>the number of tokens (in wei) rewarded to the listing per reward period</p> |
| stakedBalance | <code>BigNumber</code> | <p>the number of tokens staked by the listing owner (in wei)</p> |
| status | <code>number</code> | <p>the number representing the listing status (0: no listing, 1: proposal, 2: validator, 3: in-challenge, 4: exiting)</p> |
| tendermintPublicKey | <code>string</code> | <p>the 32 byte Tendermint public key of the listing holder</p> |

<a name="Vote"></a>

## Vote
<p>Represents a stored vote in a challenge poll.</p>

**Kind**: global typedef  
**Properties**

| Name | Type | Description |
| --- | --- | --- |
| id | <code>BigNumber</code> | <p>the challengeId the vote is for</p> |
| value | <code>string</code> | <p>the vote value (should be &quot;1&quot; or &quot;0&quot; for challenge votes)</p> |
| salt | <code>string</code> | <p>a secret string used to hash the vote; must use same salt in commit as reveal</p> |
| encoded | <code>string</code> | <p>the encoded vote, as passed to the contract system</p> |

