module health_data_sharing::consent_manager {
    use std::string::String;
    use std::signer;
    use aptos_std::table::{Self, Table};
    
    /// Represents consent for a specific data type
    struct Consent has store, drop {
        /// Who the data is shared with (recipient address)
        recipient: address,
        /// Type of health data being shared (e.g., "medications", "vitals", "lab_results")
        data_type: String,
        /// Whether consent is currently active
        is_active: bool,
        /// Optional expiration timestamp
        expiration_time: u64
    }
    
    /// Resource stored under a user's account
    struct UserConsents has key {
        /// Mapping from consent_id to Consent
        consents: Table<u64, Consent>,
        /// Counter for generating unique consent IDs
        next_consent_id: u64
    }
    
    /// Error codes
    const E_NOT_AUTHORIZED: u64 = 1;
    const E_CONSENT_NOT_FOUND: u64 = 2;
    const E_CONSENT_EXPIRED: u64 = 3;
    
    /// Initialize user's consent storage
    public entry fun initialize(account: &signer) {
        let user_addr = signer::address_of(account);
        if (!exists<UserConsents>(user_addr)) {
            move_to(account, UserConsents {
                consents: table::new(),
                next_consent_id: 0
            });
        }
    }
    
    /// Grant consent to share specific health data with a recipient
    public entry fun grant_consent(
        account: &signer,
        recipient: address,
        data_type: String,
        expiration_time: u64
    ) acquires UserConsents {
        let user_addr = signer::address_of(account);
        
        // Get or initialize user's consent storage
        if (!exists<UserConsents>(user_addr)) {
            initialize(account);
        };
        
        let user_consents = borrow_global_mut<UserConsents>(user_addr);
        let consent_id = user_consents.next_consent_id;
        
        // Create and store new consent
        table::add(&mut user_consents.consents, consent_id, Consent {
            recipient,
            data_type,
            is_active: true,
            expiration_time
        });
        
        // Increment consent ID counter
        user_consents.next_consent_id = consent_id + 1;
    }
    
    /// Revoke previously granted consent
    public entry fun revoke_consent(
        account: &signer,
        consent_id: u64
    ) acquires UserConsents {
        let user_addr = signer::address_of(account);
        assert!(exists<UserConsents>(user_addr), E_NOT_AUTHORIZED);
        
        let user_consents = borrow_global_mut<UserConsents>(user_addr);
        assert!(table::contains(&user_consents.consents, consent_id), E_CONSENT_NOT_FOUND);
        
        // Simply remove the consent entry
        table::remove(&mut user_consents.consents, consent_id);
    }
}
