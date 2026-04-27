# CometBFT: Delayed Commits 구현

> 이 문서는 CometBFT 합의 엔진에서 수행한 변경 사항을 설명합니다.
> 전체 프로젝트 개요는 [mini-core-deck 문서](https://github.com/lapidix/mini-core-deck/blob/main/docs/delayed-commits-improvement.md)를 참고하세요.

## 변경 목적

`timeout_commit`을 최소화(1ms)해도 지연 도착한 precommit 투표를 수집할 수 있도록 합의 엔진을 확장한다. 수집된 데이터는 ABCI `RequestFinalizeBlock`의 `delayed_commits` 필드를 통해 애플리케이션 레이어(Cosmos SDK)로 전달된다.

## 변경 내역

### 1. ABCI Proto 확장

**파일:** `proto/tendermint/abci/types.proto`

`RequestFinalizeBlock`에 `delayed_commits` 필드(field number 9)를 추가했다.

```protobuf
message RequestFinalizeBlock {
  repeated bytes       txs                 = 1;
  CommitInfo           decided_last_commit = 2 [(gogoproto.nullable) = false];
  repeated Misbehavior misbehavior         = 3 [(gogoproto.nullable) = false];
  bytes                hash                = 4;
  int64                height              = 5;
  google.protobuf.Timestamp time           = 6 [(gogoproto.nullable) = false, (gogoproto.stdtime) = true];
  bytes                next_validators_hash = 7;
  bytes                proposer_address    = 8;
  CommitInfo           delayed_commits     = 9 [(gogoproto.nullable) = false]; // 신규
}
```

- 기존 `CommitInfo` 타입 재활용 → 하위 호환성 유지
- `gogoproto.nullable = false` → 값 타입으로 생성 (포인터 아님)
- `abci/types/types.pb.go`에 struct 필드, getter, marshal/unmarshal, size 수동 반영

### 2. 합의 엔진 — DelayedPrecommits 버퍼

**파일:** `consensus/state.go`

#### State 구조체 확장

```go
type State struct {
    // ... 기존 필드 ...

    // DelayedPrecommits stores precommit votes for the previous height that arrived
    // after RoundStepNewHeight ended.
    DelayedPrecommits *types.VoteSet
}
```

#### addVote — 지연 precommit 수집

기존에는 `RoundStepNewHeight` 이후 도착한 이전 height의 precommit을 무시했다:

```go
// 변경 전 (consensus/state.go ~L2138)
if cs.Step != cstypes.RoundStepNewHeight {
    cs.Logger.Debug("precommit vote came in after commit timeout and has been ignored")
    return added, err
}
```

변경 후 `DelayedPrecommits` 버퍼에 수집:

```go
// 변경 후
if cs.Step != cstypes.RoundStepNewHeight {
    if cs.DelayedPrecommits != nil {
        added, err = cs.DelayedPrecommits.AddVote(vote)
        if added {
            cs.Logger.Debug("added late precommit to delayed buffer",
                "vote_height", vote.Height, "current_height", cs.Height)
        }
    } else {
        cs.Logger.Debug("precommit vote came in after commit timeout and has been ignored")
    }
    return added, err
}
```

이 변경은 `if vote.Height+1 == cs.Height && vote.Type == PrecommitType` 블록 내부의 `cs.Step != RoundStepNewHeight` 분기만 수정한다. 기존 `RoundStepNewHeight` 동안의 투표 수집 경로(`cs.LastCommit.AddVote`)는 전혀 변경하지 않는다.

#### updateToState — 버퍼 초기화

`updateToState`에서 새 height로 전환 시 버퍼를 초기화:

```go
// LastCommit 설정 직후
if state.LastBlockHeight > 0 && cs.CommitRound > -1 {
    cs.DelayedPrecommits = types.NewVoteSet(
        state.ChainID, state.LastBlockHeight,
        cs.CommitRound, cmtproto.PrecommitType, state.LastValidators,
    )
} else {
    cs.DelayedPrecommits = nil
}
```

#### finalizeCommit — BlockExecutor에 전달

`ApplyVerifiedBlock` 호출 직전에 버퍼를 BlockExecutor에 전달:

```go
cs.blockExec.SetDelayedPrecommits(cs.DelayedPrecommits)
```

### 3. 블록 실행 — delayed_commits 조립

**파일:** `state/execution.go`

#### BlockExecutor 확장

```go
type BlockExecutor struct {
    // ... 기존 필드 ...
    delayedPrecommits *types.VoteSet
}

func (blockExec *BlockExecutor) SetDelayedPrecommits(votes *types.VoteSet) {
    blockExec.delayedPrecommits = votes
}
```

#### BuildDelayedCommitInfo

VoteSet과 ValidatorSet을 받아 `abci.CommitInfo`를 조립:

```go
func BuildDelayedCommitInfo(delayedVotes *types.VoteSet, lastValSet *types.ValidatorSet) abci.CommitInfo {
    if delayedVotes == nil || lastValSet == nil {
        return abci.CommitInfo{}
    }
    votes := make([]abci.VoteInfo, len(lastValSet.Validators))
    for i, val := range lastValSet.Validators {
        blockIDFlag := cmtproto.BlockIDFlagAbsent
        if vote := delayedVotes.GetByAddress(val.Address); vote != nil {
            if vote.BlockID.IsComplete() {
                blockIDFlag = cmtproto.BlockIDFlagCommit
            } else {
                blockIDFlag = cmtproto.BlockIDFlagNil
            }
        }
        votes[i] = abci.VoteInfo{
            Validator:   types.TM2PB.Validator(val),
            BlockIdFlag: blockIDFlag,
        }
    }
    return abci.CommitInfo{Round: delayedVotes.Round(), Votes: votes}
}
```

#### applyBlock — FinalizeBlock 요청에 포함

```go
abciResponse, err := blockExec.proxyApp.FinalizeBlock(ctx, &abci.RequestFinalizeBlock{
    // ... 기존 필드 ...
    DelayedCommits: blockExec.buildDelayedCommits(block, state), // 신규
})
```

## 데이터 흐름

```
Height H 합의 완료:
  finalizeCommit(H)
    ├→ seenCommit 저장 (blockStore)
    ├→ ApplyVerifiedBlock (ABCI FinalizeBlock 호출)
    │    └→ DelayedCommits: 이전 height의 지연 투표
    ├→ updateToState
    │    ├→ cs.LastCommit = Votes.Precommits(CommitRound)  [라이브 참조]
    │    └→ cs.DelayedPrecommits = new VoteSet for H       [버퍼 초기화]
    └→ scheduleRound0 (timeout_commit 만큼 대기)

  [timeout_commit 대기 동안]
    └→ addVote: vote.Height+1 == cs.Height
        ├→ RoundStepNewHeight → cs.LastCommit.AddVote()    [기존 경로]
        └→ 그 외 → cs.DelayedPrecommits.AddVote()          [신규 경로]

Height H+1:
  enterPropose
    └→ cs.LastCommit.MakeExtendedCommit() → block.LastCommit
  finalizeCommit(H+1)
    └→ FinalizeBlock { DelayedCommits: H의 지연 투표 }
```

## 커밋 히스토리

```
ac2205a feat(abci): add delayed_commits field to RequestFinalizeBlock
5c955fe feat(consensus): add DelayedPrecommits buffer and collect late precommits
36a0163 feat(state): build and pass delayed_commits in FinalizeBlock request
```

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| `proto/tendermint/abci/types.proto` | `delayed_commits` 필드(9) 추가 |
| `abci/types/types.pb.go` | proto 재생성 (수동) |
| `consensus/state.go` | `DelayedPrecommits` 필드, `addVote` 확장, `updateToState` 초기화, `finalizeCommit` 전달 |
| `state/execution.go` | `delayedPrecommits` 필드, `SetDelayedPrecommits`, `BuildDelayedCommitInfo`, `buildDelayedCommits`, `applyBlock` 수정 |
