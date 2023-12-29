# 作业一 POE 存证测试用例

## 测试 Runtime

```rust
use crate as pallet_poe;
use frame_support::parameter_types;
use frame_system as system;
use sp_core::H256;
use sp_runtime::{
	testing::Header,
	traits::{BlakeTwo256, IdentityLookup},
};

type UncheckedExtrinsic = frame_system::mocking::MockUncheckedExtrinsic<Test>;
type Block = frame_system::mocking::MockBlock<Test>;

frame_support::construct_runtime!(
	pub enum Test where
		Block = Block,
		NodeBlock = Block,
		UncheckedExtrinsic = UncheckedExtrinsic,
	{
		System: frame_system::{Pallet, Call, Config, Storage, Event<T>},
		PoeModule: pallet_poe::{Pallet, Call, Storage, Event<T>},
	}
);

parameter_types! {
	pub const BlockHashCount: u64 = 250;
	pub const SS58Prefix: u8 = 42;
	pub const MaximumClaimLength: u32 = 512;
	pub const MinimumClaimLength: u32 = 32;
}

impl system::Config for Test {
	type BaseCallFilter = frame_support::traits::Everything;
	type BlockWeights = ();
	type BlockLength = ();
	type DbWeight = ();
	type Origin = Origin;
	type Call = Call;
	type Index = u64;
	type BlockNumber = u64;
	type Hash = H256;
	type Hashing = BlakeTwo256;
	type AccountId = u64;
	type Lookup = IdentityLookup<Self::AccountId>;
	type Header = Header;
	type Event = Event;
	type BlockHashCount = BlockHashCount;
	type Version = ();
	type PalletInfo = PalletInfo;
	type AccountData = ();
	type OnNewAccount = ();
	type OnKilledAccount = ();
	type SystemWeightInfo = ();
	type SS58Prefix = SS58Prefix;
	type OnSetCode = ();
}

impl pallet_poe::Config for Test {
	type Event = Event;
	type MaximumClaimLength = MaximumClaimLength;
	type MinimumClaimLength = MinimumClaimLength;
}

// Build genesis storage according to the mock runtime.
pub fn new_test_ext() -> sp_io::TestExternalities {
	system::GenesisConfig::default().build_storage::<Test>().unwrap().into()
}

```

## 创建存证的测试用例

代码片段：

```rust
#[test]
fn create_claim_failed_when_too_big() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; (MaximumClaimLength::get() + 1).try_into().unwrap()];
		assert_noop!(
			PoeModule::create_claim(Origin::signed(1), claim.clone()),
			Error::<Test>::ClaimTooBig
		);
	});
}

#[test]
fn create_claim_failed_when_too_small() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; (MinimumClaimLength::get() - 1).try_into().unwrap()];
		assert_noop!(
			PoeModule::create_claim(Origin::signed(1), claim.clone()),
			Error::<Test>::ClaimTooSmall
		);
	});
}

#[test]
fn create_claim_works() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		assert_ok!(PoeModule::create_claim(Origin::signed(1), claim.clone()));
		assert_eq!(
			Proofs::<Test>::get(&claim),
			Some((1, frame_system::Pallet::<Test>::block_number()))
		)
	});
}
```

## 撤销存证测试用例

代码片段：

```rust
#[test]
fn revoke_claim_works() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		let _ = PoeModule::create_claim(Origin::signed(1), claim.clone());
		assert_ok!(PoeModule::revoke_claim(Origin::signed(1), claim.clone()));
		assert_eq!(Proofs::<Test>::get(&claim), None)
	});
}

#[test]
fn revoke_claim_failed_when_claim_is_not_exist() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		assert_noop!(
			PoeModule::revoke_claim(Origin::signed(1), claim.clone()),
			Error::<Test>::ClaimNotExist
		);
	})
}

#[test]
fn revoke_claim_failed_when_sender_is_not_owner() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		let _ = PoeModule::create_claim(Origin::signed(1), claim.clone());
		assert_noop!(
			PoeModule::revoke_claim(Origin::signed(2), claim.clone()),
			Error::<Test>::NotClaimOwner
		);
	}
```

## 转移存证的测试用例

代码片段：

```rust
#[test]
fn transfer_claim_works() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		let _ = PoeModule::create_claim(Origin::signed(1), claim.clone());
		assert_ok!(PoeModule::transfer_claim(Origin::signed(1), 3, claim.clone()));
		assert_eq!(
			Proofs::<Test>::get(&claim),
			Some((3, frame_system::Pallet::<Test>::block_number()))
		)
	});
}

#[test]
fn transfer_claim_failed_when_claim_is_not_exist() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		assert_noop!(
			PoeModule::transfer_claim(Origin::signed(1), 1, claim.clone()),
			Error::<Test>::ClaimNotExist
		);
	})
}

#[test]
fn transfer_claim_failed_when_sender_is_not_owner() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		let _ = PoeModule::create_claim(Origin::signed(1), claim.clone());
		assert_noop!(
			PoeModule::transfer_claim(Origin::signed(3), 2, claim.clone()),
			Error::<Test>::NotClaimOwner
		);
	})
}

#[test]
fn transfer_claim_failed_when_sender_is_destination() {
	new_test_ext().execute_with(|| {
		let claim = vec![1; 64];
		let _ = PoeModule::create_claim(Origin::signed(1), claim.clone());
		assert_noop!(
			PoeModule::transfer_claim(Origin::signed(1), 1, claim.clone()),
			Error::<Test>::DestinationIsClaimOwner
		);
	})
}

```
