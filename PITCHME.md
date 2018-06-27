### VCR pros and cons

---

### TIL

- Our current mocking strategy & why it sucks
- What is VCR
- Why it is better
- Why it is worse

---

### Background

```ruby
register_stub('get', %r{.*/countries$}, { countries: YAMLLoader.get_test_data('countries.yml')['countries'] }.to_json)
register_stub('get', %r{.*/states$}, { states: YAMLLoader.get_test_data('states.yml')['states'] }.to_json)
register_stub('get', %r{.*/ride_states$}, { ride_states: YAMLLoader.get_test_data('ride_states.yml')['states'] }.to_json)
register_stub('get', %r{.*/reasons_for_delay$}, { reasons_for_delay: YAMLLoader.get_test_data('reasons_for_delay.yml')['reasons'] }.to_json)
register_stub('get', %r{.*/case_closure_reason_codes$}, { case_closure_reason_codes: YAMLLoader.get_test_data('case_closure_reason_codes.yml')['case_closure_reason_codes'] }.to_json)
register_stub('get', %r{.*/citizenship_statuses}, { citizenship_statuses: YAMLLoader.get_test_data('citizenship_statuses.yml')['citizenship_statuses'] }.to_json)
register_stub('post', %r{.*/self_lock$}, {}.to_json)
register_stub('get', %r{.*/coa_codes$}, { coa_codes: YAMLLoader.get_reference_data('coa_codes.yml')['sevis'].map { |c| { code: c } }.concat([{ code: '12A' }]) }.to_json)
register_stub('get', %r{.*/us_states.*$}, {}.to_json)
```

---

### Cons

- Regex matching
- Order dependent
- What we're returning != reality
- Reconcile response hashes as our code changes
- Silent failures

---

### VCR

Records your test suite's HTTP interactions and replays them during future test runs for fast, deterministic, accurate tests.

---

```ruby
context 'when valid response' do
  let(:persisted_ids) { [2] }

  before do
    register_stub(:get, %r{.*/read/case_headers\/?.*$}, { case_header_ids: persisted_ids }.to_json)
  end

  it 'returns the subset of persisted case ids' do
    case_header_ids = [1, 2, 3]
    query           = CasePersistenceQuery.new(case_header_ids: case_header_ids)
    query.execute

    expect(query.response).to eq(persisted_ids)
  end
end
```

@[4-6](Register a stub whenever an HTTP request is made)

+++

```ruby
context 'when valid response', :vcr do
  let(:persisted_ids) { [2] }

  it 'returns the subset of persisted case ids' do
    case_header_ids = [1, 2, 3]
    query           = CasePersistenceQuery.new(case_header_ids: case_header_ids)
    query.execute

    expect(query.response).to eq(persisted_ids)
  end
end
```

@[1](Use RSpec metadata to let VCR know this test makes external requests)

+++

```yml
# spec/cassettes/CasePersistenceQuery/_execute/when_valid_response/returns_the_subset_of_persisted_case_ids.yml

http_interactions:
- request:
    method: get
    uri: https://nonprod-eve-cases-data.vis.nonprod.uscis.dhs.gov/read/case_headers?case_header_ids%5B%5D=1&case_header_ids%5B%5D=2&case_header_ids%5B%5D=3
    body:
      string: '{"case_header_ids":[1,2,3]}'
    headers: ...redacted
  response:
    status:
      code: 200
      message: OK
    headers: ...redacted
    body:
      string: '{"case_header_ids":[2]}'
```

@[1](File path is intuitive)
@[4-9](Exact request is recorded)
@[10-16](Response oftentimes just works, or you can modify your stub here)

---

### Pros of VCR

- Minimal setup
- No management of stubs
- Optionally re-records cassettes to keep them fresh and current.
- Dynamic response files

---

### Cons of VCR

- Pollute git with cassettes
