(ns souji.design.operation-test
  (:require [clojure.test :refer [deftest is testing]]
            [kotoba.lang.text :as str]
            [souji.design.operation :as op]))

(deftest ops-are-declared-design-only
  (testing "five design ops, none of them actuates hardware"
    (is (= #{:design/propose :design/validate :safety/gate :sim/run :bom/observe}
           op/op-names))
    (is (op/observable? :bom/observe))
    (is (not (op/observable? :design/propose)))))

(deftest actuator-paths-are-absent
  (testing "no op drives a robot: actuator paths must never exist here"
    (is (not (contains? op/op-names :robot/command)))
    (is (not (contains? op/op-names :actuator/drive)))
    (is (thrown? clojure.lang.ExceptionInfo (op/propose :robot/command {})))))

(deftest unknown-ops-are-refused
  (testing "no ambient ops: an unadmitted op name throws"
    (is (thrown? clojure.lang.ExceptionInfo (op/propose :bom/price {})))
    (is (thrown? clojure.lang.ExceptionInfo (op/govern {:proposal/op :bom/price})))))

(deftest propose-never-executes
  (testing "proposal comes back pending-govern, not done"
    (let [p (op/propose :design/validate {:modelled true})]
      (is (= :pending-govern (:proposal/status p)))
      (is (= :propose (:proposal/effect p))))))

(deftest observe-ops-complete
  (testing ":bom/observe is a pure observation and completes in-govern"
    (let [p (op/propose :bom/observe {:parts []})]
      (is (= :done (:proposal/status (op/govern p)))))))

(deftest safety-gates-are-mandatory-in-design-proposals
  (testing "a design proposal without both safety gates is refused"
    (is (thrown? clojure.lang.ExceptionInfo
                 (op/govern (op/propose :design/propose
                                        {:mechanism {:brush-count 2}
                                         :safety/gates {}})))))
  (testing "a design proposal with both gates passes"
    (let [p (op/propose :design/propose
                        {:mechanism {:brush-count 2}
                         :safety/gates {:child-pet-contact-stop true
                                        :sharp-object-stop true}})]
      (is (= :approved-for-review (:proposal/status (op/govern p)))))))

(deftest sim-results-are-modelled-not-measured
  (testing "sim/run results carry the modelled label — no measured data exists"
    (let [p (op/propose :sim/run {:floor-area 20 :obstacles 5 :battery 3000})
          g (op/govern p)]
      (is (= :approved-for-review (:proposal/status g)))
      (is (= :propose (:proposal/effect g)) "sim/run is a propose op")
      (is (= "R0: propose-only; actuation path does not exist here" (:proposal/note g))
          "govern note reminds that no actuation happens (design-only)"))))