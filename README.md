# Junto Holochain Traits

The following paragraphs describe several Holochain DNA traits _(not Rust traits - Rust syntax only used to have syntax highlighting)_ that are used to interface different components of the Holochain based Junto back-end with its UI.

The main intention behind defining traits as abstract interfaces and hence introducing a contract that the UI can follow without being coupled to the implemention of those traits is clearly to decouple zome code and UI code such that the Junto Holochain back-end can be implemented by several micro-service DNAs with clear concerns and which can be exchanged or extended by 3rd party DNAs later on.

Special emphasis lies on the `Expression` trait which enables Junto users to define new languages of posts and messages by writing a DNA.
Because of the agent-centric nature of every Holochain app, every DNA can be regarded as dealing with expressions: every entry exists because some agent _speaks it into being_.
By choosing to add implementations of all `Expression` trait functions, an already existing DNA can be retrofitted to be shown within Junto - this renders Junto a kind of agent-centric browser for a wide variety of agent-centric hApps/DNAs.

...

## Social Graph

```rust
use dpki::DpkiRootHash;
type Identity = DpkiRootHash;

/// Trait that provides an interface for creating and maintaining a social graph
/// between agents.
///
/// Follows & Connections between agents can take an optional
/// metadata parameter; "by".
/// This parameter is used to associate some semantic between relationships.
/// In Junto's case this field could be leveraged to create
/// user definable perspectives. Example; follow graph for my:
/// holochain connections, personal connections and drone connections
///
/// The other possibility is to create a new DNA implementing this trait
/// for each social graph context the user wants to define.
pub trait SocialGraph {
    // Follow Related Operations
    fn my_followers(by: Option<String>) -> ExternResult<Vec<Identity>>;
    fn followers(followed_agent: Identity, by: Option<String>) -> ExternResult<Vec<Identity>>;
    fn nth_level_followers(
        n: usize,
        followed_agent: Identity,
        by: Option<String>,
    ) -> ExternResult<Vec<Identity>>;

    fn my_followings(by: Option<String>) -> ExternResult<Vec<Identity>>;
    fn following(following_agent: Identity, by: Option<String>) -> ExternResult<Vec<Identity>>;
    fn nth_level_following(
        n: usize,
        following_agent: Identity,
        by: Option<String>,
    ) -> ExternResult<Vec<Identity>>;

    fn follow(target_agent: Identity, by: Option<String>) -> ExternResult<()>;
    fn unfollow(target_agent: Identity, by: Option<String>) -> ExternResult<()>;

    // Connection Related Operations (i.e. bidirectional friendship)
    fn my_friends() -> ExternResult<Vec<Identity>>;
    fn friends_of(agent: Identity) -> ExternResult<Vec<Identity>>;

    fn request_friendship(target_agent: Identity) -> ExternResult<()>;
    fn decline_friendship(target_agent: Identity) -> ExternResult<()>;

    fn incoming_friendship_requests() -> ExternResult<Vec<Identity>>;
    fn outgoing_friendship_requests() -> ExternResult<Vec<Identity>>;

    fn drop_friendship(target_agent: Identity) -> ExternResult<()>;
}
```

## Social Context

```rust
/// Entry marking a reference to some other entry in another DNA
#[derive(Serialize, Deserialize, Clone)]
pub struct GlobalEntryRef {
    pub dna: HoloHash<Dna>,
    pub entry_address: HoloHash<Header>,
}

/// Trait that provides an interface for associating entries in foreign DNA's to a social context/collective.
/// The social context is not something explictly interfaceable via the trait but instead something
/// which is infered based on the collective the DNA is serving.
///
/// It is important that the collective should not be thought of as just a group - it can instead be thought of as the root
/// between social communication over some shared perspective regardless of the method of communication (expression).
/// This shared perspective could be private and "defined" in the case of a group but can also
/// be something more generic and public such as a topic of conversation or even a time.
///
/// This also means that the social context can be fractal. So for example you could have a group protected by membrane rules
/// which contains a comment_link to another social context which is a topic of communication within that group which is
/// also protected by the same membrane rules.
///
/// Same logic can also be applied for public topic based scenarios, where moderators/members of topic
/// ( dependant on configuration of host DNA ) could register sub/sibling topics and groups as a fractal social context.
///
/// If a social context desires privacy; the host DNA should be membraned along with any other DNA's which is reference by this DNA
pub trait SocialContext {
    /// Persist to social context that you have made an entry at expression_ref.dna_address/@expression_ref.entry_address
    /// which is most likely contextual to the collective of host social context
    fn post(expression_ref: GlobalEntryRef) -> ExternResult<()>;
    /// Register that there is some dna at dna_address that you are communicating in.
    /// Others in collective can use this to join you in new DNA's
    fn register_communication_method(dna_address: HoloHash<Dna>) -> ExternResult<()>;
    /// Is current agent allowed to write to this DNA
    fn writable() -> bool;
    /// Get GlobalEntryRef for collective; queryable by dna or agent or all. DHT hotspotting @Nico?
    fn read_communications(
        by_dna: Option<HoloHash<Dna>>,
        by_agent: Option<Identity>,
        count: usize,
        page: usize,
    ) -> ExternResult<Vec<GlobalEntryRef>>;
    /// Get DNA's this social context is communicating in
    fn get_communication_methods(count: usize, page: usize) -> ExternResult<Vec<HoloHash<Dna>>>;
    /// Get agents who are a part of this social context
    /// optional to not force every implementation to create a global list of members - might be ok for small DHTs
    fn members(count: usize, page: usize) -> ExternResult<Option<Vec<Identity>>>;
}
```

## Expressions

```rust
/// A holochain expression
pub struct Expression {
    pub expression: Element,
    pub expression_dna: HoloHash<Dna>,
}

/// An interface into a DNA which contains Expression information. Expected to be interacted with using expression Addresses
/// retrieved from a social context or by using a Identity retreived from a users social graph.
/// In this situation you can see that the Expression DNA/trait does not need to include any index capability
/// as this is already infered to the agent by the place they got the expression from; social context or social graph.
///
/// If the expression should be private to a group of people then the host DNA should be membraned.
pub trait Expression {
    /// Create an expression and link it to yourself publicly with optional dna_address pointing to
    /// dna that should ideally be used for linking any comments to this expression
    fn create_public_expression(content: String) -> ExternResult<Expression>;
    /// Get expressions authored by a given Agent/Identity
    fn get_by_author(
        author: Identity,
        page_size: usize,
        page_number: usize,
    ) -> ExternResult<Vec<Expression>>;
    fn get_expression_by_address(address: AnyDhtHash) -> ExternResult<Option<Expression>>;

    /// Send an expression to someone privately p2p
    fn send_private(to: Identity, content: String) -> ExternResult<String>;
    /// Get private expressions sent to you optionally filtered by sender address
    fn inbox(
        from: Option<Identity>,
        page_size: usize,
        page_number: usize,
    ) -> ExternResult<Vec<Expression>>;
}
```

## Inter-DNA

```rust
/// Entry marking a reference to some other entry in another DNA
#[derive(Serialize, Deserialize, Clone)]
pub struct GlobalEntryRef {
    pub dna: HoloHash<Dna>,
    pub entry_address: HoloHash<Header>,
}

/// Interface for cross DNA links. Allows for the discovery of new DNA's/entries from a known source DNA/entry.
/// Host DNA should most likely implement strong anti spam logic if this is to be a public - unmembraned DNA.
pub trait InterDNA {
    fn create_link(source: GlobalEntryRef, target: GlobalEntryRef) -> ExternResult<()>;
    fn remove_link(source: GlobalEntryRef, target: GlobalEntryRef) -> ExternResult<()>;

    fn get_outgoing(source: GlobalEntryRef, count: usize, page: usize) -> ExternResult<Vec<GlobalEntryRef>>;
    fn get_incoming(target: GlobalEntryRef, count: usize, page: usize) -> ExternResult<Vec<GlobalEntryRef>>;
}

```
