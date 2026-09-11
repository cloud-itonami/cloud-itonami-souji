(ns souji.design.operation)

;; Cleaning Robot Design Governor ⊣ Advisor — R0 skeleton.
;; Design-only: every op is :propose or :observe. There is NO op that drives
;; a physical robot. Actuator command paths must never be added to this repo.
;; Safety gates (child/pet contact stop, sharp-object stop) are permanent
;; structural invariants — they are never absent from any phase's :auto set.

(def ops
  "Design-plane ops. Nothing here actuates hardware."
  [{:op/name :design/propose   :op/effect :propose}
   {:op/name :design/validate  :op/effect :propose}
   {:op/name :safety/gate      :op/effect :propose}
   {:op/name :sim/run          :op/effect :propose}
   {:op/name :bom/observe      :op/effect :observe}])

(def op-names
  (set (map :op/name ops)))

(defn observable?
  "Only :bom/observe is a pure observation. Everything else must round-trip
   through the governor."
  [op-name]
  (= op-name :bom/observe))

(defn admitted-op?
  [op-name]
  (contains? op-names op-name))

(defn safety-gate-invariants
  "Permanent structural invariants. The governor refuses to lower them:
   child/pet contact stop and sharp-object stop exist in every proposed
   design. If a proposal lacks them, the gate fails."
  [proposal]
  (let [payload (:proposal/payload proposal {})
        gates (:safety/gates payload {})]
    (every? #(contains? gates %) [:child-pet-contact-stop :sharp-object-stop])))

(defn propose
  "Advisor side: build a design proposal. R0 returns the proposal verbatim;
   the governor reviews it. Never executes, never actuates."
  [op-name payload]
  (when-not (admitted-op? op-name)
    (throw (ex-info "op not admitted" {:op/name op-name :admitted op-names})))
  {:proposal/op     op-name
   :proposal/effect (get (into {} (map (juxt :op/name :op/effect) ops)) op-name)
   :proposal/payload payload
   :proposal/status  :pending-govern})

(defn govern
  "Governor side: review a proposal. Safety gates must be present — a design
   proposal missing child/pet stop or sharp-object stop is refused outright.
   R0 policy: :propose ops return :approved-for-review (execution path does
   not exist); :observe ops complete."
  [{:proposal/keys [op effect] :as proposal}]
  (when-not (admitted-op? op)
    (throw (ex-info "op not admitted" {:op/name op :admitted op-names})))
  (when (= op :design/propose)
    (when-not (safety-gate-invariants proposal)
      (throw (ex-info "safety gates are mandatory: child-pet-contact-stop and
                       sharp-object-stop must be present in every design proposal"
                      {:proposal/op op
                       :missing     (remove #(contains? (get-in proposal [:proposal/payload :safety/gates] {}) %)
                                            [:child-pet-contact-stop :sharp-object-stop])}))))
  (if (= effect :observe)
    (assoc proposal :proposal/status :done)
    (assoc proposal :proposal/status :approved-for-review
                    :proposal/note "R0: propose-only; actuation path does not exist here")))